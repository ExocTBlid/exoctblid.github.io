---
layout: post
title:  "Cluster Roll"
date:   2018-03-05T14:59:42-07:00
categories: python aws
---
This script does the majority of the work to restart the Tomcat application of a cluster in AWS EC2 split by availability zone.

{% highlight Python %}
#!/usr/bin/python
import os
import subprocess
from glob import glob
import time
import sys
from lib import util, ec2
import datetime
import click


@click.command()
@click.option('-e', '--environment', type=click.Choice(['adm', 'stg', 'prd']), default='stg',
              help='name of the aws credential profile to use. (default: stg)')
@click.option('-s', '--stack', type=click.Choice(['car', 'cia', 'ctva', 'shared', 'skyui', 'spec', 'specu',
                                                  'stva']),
              default='spec', help='The stack: spec, shared, etc.')
@click.option('-a', '--application', help='The application name on the cluster.')
@click.option('-n', '--node', help="set the aws authentication node (ex: i-8756432)")
@click.option('-f', '--fullname', help="use the fullname of the node (ex: prd-car-mango-i-a1649f5e)")
@click.option('-r', '--reason', help="passing the reason why it was restarted.")
def main(environment, stack, application, node, fullname, reason):
    if fullname:
        parts = fullname.split('-')
        environment = parts[0]
        stack = parts[1]
        application = parts[2]
        node = "i-{}".format(parts[4])
    if application is None and node is None:
        print("\nYou must specify either an application, a node, or both... so we don't restart the world.\n")
        help()
        exit(1)
    config = util.load_config(environment)
    conn = ec2.connect(config)
    search_string = util.build_search_string(environment=environment, stack=stack,
                                             application=application, node=node)
    print("Search String: " + search_string)
    reservations = conn.get_all_reservations(filters={"tag:Name": search_string})
    instances = [i for r in reservations for i in r.instances]
    if len(instances) < 1:
        print("No nodes found.")
        sys.exit(1)
    else:
        print("Instances found: " + str(len(instances)))

    print("\nSelected Nodes:")
    eureka_statuses = {}
    for instance in instances:
        instance_name = instance.tags['Name']
        if util.roll_anyway(instance):
            print "Roll Anyway"
            eureka_status = "UP"
        else:
            print "Normal Roll Check"
            eureka_status = util.eureka_status(config['eureka_url'], instance)
        eureka_statuses[instance_name] = eureka_status
        print "{} ({})".format(instance_name, eureka_status)

    print("\nThe above nodes will have their service restarted if NOT disabled.")
    print("Waiting 10 seconds to give you a chance to abort.\n")
    time.sleep(10)

    for f in glob("hosts/us*"):
        os.remove(f)

    if os.environ.get('BUILD_NUMBER') is not None:
        build = os.environ.get('BUILD_NUMBER')
    else:
        build = 'NA'

    hosts = {}
    for instance in instances:
        instance_name = instance.tags['Name']
        eureka_status = eureka_statuses[instance_name]
        service = util.get_service(instance.tags['Name'])

        if service == 'tomcat':
            if eureka_status != 'UP':
                if reason != 'Chef Deploy':
                    print("Skipping Node")
                    continue

        print(instance.update())
        if instance.update() == 'running':
            ip = instance.interfaces[0].private_ip_address
            az = instance.placement
            file = "{}-{}".format(az, service)
            stack = instance.tags['Name'].split('-')[1]
            eureka_url = util.get_eureka_url(config['eureka_url'], instance)
            if file not in hosts:
                hosts[file] = []
            hosts[file].append(ip + " eureka_url=" + eureka_url + " eureka_status=" + eureka_status + " stack=" + stack)
            with open('/var/log/jenkins/rolling.log', 'ab') as log_f:
                print("Adding hostname to file")
                log_f.write("{} | {} ({}) | {} | {} | \"{}\"\r\n".format(datetime.datetime.now(), instance_name,
                                                                    eureka_status, ip, build, reason))

    for key, value in hosts.iteritems():
        print key
        f = open('hosts/' + key, 'w')
        f.write("\n".join(value))
        f.close()

    print("Starting restarts...")
    for f in glob("hosts/us*"):
        subprocess.call(["echo", f])
        subprocess.call(["/usr/local/bin/ansible-playbook", "-i" + f, "restart_" + service + ".yml"])
    for instance in instances:
        util.send_to_graphite(environment, instance)


if __name__ == "__main__":
    main()
{% endhighlight %}

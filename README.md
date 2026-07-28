

# ![FPA - Export](/al/altihex-FPA/Documentation/README/FPA - Export.svg)    Forward Proxy Appliance

## Why use it?

- You want to improve your security posture by reducing your attack surface when an EC2 instance, ECS container or EKS Pod are compromised by an attacker.  

- You want an AWS aware configuration that doesn’t require manual intervention for every instance, container or pod startup.

- You want to ensure the proxy configuration is not configurable from within your application accounts.

- You want to be able to configure VPCs with overlapping CIDR blocks.

- You don’t want any peering limits.

- You want support for http as well as https. 

- You want support for different ports and transports 

## What is it? 

- This is a forward proxy appliance that is configurable using host groups and domains in a separate security account behind a service endpoint and gateway loadbalancer. Instance, containers and pods are configured with metadata tags to select host profiles for internet access.

- Each application account will have their private outbound networked appliance traffic routed through the service endpoint for filtering. The proxy is transparent.  

- The proxy is low maintenance as host profiles can be tagged or in the case of pods annotated to a specific host profile to allow access to curated list of domains.

- When an instance, container or pod starts up for the first time a simple notification Api can be used to configure the proxy to allow outbound connections. This allows for automated access based on profiles. 

- The appliances are clustered and with minimal configuration will gather configuration information across all your accounts. The cluster manages this from a single node reducing the number of AWS api requests and allowing horizontal cluster scaleability. 

## How is this usually addressed?

- Block all outbound access and manually configure security group rules for a selected websites. For common site access to enable automated patching for example push access to monitoring and logging sites configure this for each specific instance type. Isolate the instances to specific security groups and manage the instances and security group mapping manually. This approach is difficult and time intensive to maintain also prone to errors. 

- Configure a security group to allow access to all outbound traffic. For a compromised instance, container or pod, this allows an attacker to phone home and orchestrate an attack on your application. 

## Other less commonly used approaches

- Use Squid and manually configure each subnet and host name. This will be either too wide or require intensive amounts of labour. Most likely for EC2 Auto Scaling Groups,  ECS containers and EKS pods will be un-maintainable and will cause production issues.  

- Configure the instance firewall. If the instance is fully compromised this can be disabled by an attacker.

# Release Notes

## Support 

- Please log all support issues here - [issues](https://github.com/Altihex/Forward-Proxy-Appliance/issues)

- For further support enquiries  email: support@altihex.com


## Restrictions

- No support for the Indian market at present. This will be addressed in the future.

- Currently only the following instance types are supported for the apliance:

  - t3.large

  - t3a.large

  - m5.large

  - m5a.large


- Inbound connections are **not** supported as this is a forward proxy.

- **Only** IPv4 is supported at present. IPv6 is **not** supported with this release. It is on the road-map for a future release.


- Encrypted SNI / ECH **must** be disabled with TLS 1.3 connections. This could be a privacy issue for your users, if so this proxy is not the solution you need. 



## Basic Account Design

The appliance is run as part of a private link / endpoint service with a gateway load balancer, with multiple  consumer accounts. 



![FPA - Design](/al/altihex-FPA/Documentation/README/FPA - Design.svg)

### Consumer account 

This requires two sets of private subnets. The primary subnet will have all the instances / containers and pods deployed on them. The primary subnets will be routed to the local service endpoints using the  second subnet for the private link service using route 0.0.0.0/0. The second private subnet routes all traffic on to the NAT Gateway after it returns from the Gateway Load Balancer.


### Proxy processes


Each FPA instance needs to be uniquely tagged using the **altihex-FPA-cluster-node** key and name. The name is used for unique cloudwatch log group names. If don't set the cluster quorum to at least three, then each cluster node will be a controller.  Notifications will fail, as a single node cluster will not send out notifications to the other nodes.  The notifications API updates all the cluster nodes with the new node and updates the proxy configuration on the fly.. This is needed when new EC2, ECS or EKS nodes are started.


There are two executables, fpa and fpa-cluster.


### Fpa-cluster

This controls the cluster functionality and configuration loading and scanning. 

Each FPA instance needs to be uniquely tagged using the altihex-FPA-cluster-node key and name. The name is used for unique cloudwatch logs. If you do not tag the instances and set the quorum to more than 1. Then notifications can fail, the notifications api when sent to an fpa proxy node will allow the proxy configuration to be updated on the fly. This is needed when new EC2, ECS or EKS nodes are started.

### Healthcheck

This is a health check process for the Gateway Load Balancer. It listens on port 80 for gateway load balancer target group pings. It runs within the fpa-cluster executable. 


### Fpa

This is the forward proxy. It uses the configuration config.yaml file on startup, but uses the account scanning results from the fpa-cluster process. If the fpa-cluster stops running the fpa process will be killed as well. 


### Cloudwatch


Cloudwatch Logs are automatically created in the security account for the fpa process and the cluster process in each node, using the value of the tagged name: on the appliance.


## Configuration

Configuration of the proxy is controlled with a single configuration yaml file. With no configuration file configured the proxy will fail to start. 


This needs to put into the root directory of an S3 bucket in the security account. Default naming conventions:

- S3 bucket – **altihex-fpa-\<account\_id\>-region**

- configuration file – **config.yaml**

### EC2 instances

Set the scan tag on the EC2, with a key of **altihex-FPA-host-groups** and a value with the **host_tag_profiles** that you want to allow through the proxy. The **host\_tag\_profiles** values can be space or comma separated.

### ECS Containers

Set the scan tag on the ECS task to allow ECS containers to access the proxy. Use the key of **altihex-FPA-host-groups** and a value with the **host\_tag\_profiles** that you want to allow through the proxy. The **host\_tag\_profiles** values can be space or comma separated.

### EKS Pods

Kubernetes PODS don’t use tags, so the proxy scans annotations in the kind: Deployment, **spec: template: metadata:annotations:** key. Use a key of **altihex-FPA-host-groups** and a value with the **host\_tag\_profiles** that you want to allow through the proxy. The **host\_tag\_profiles** values can be space or comma separated.

Due to the non AWS nature of kubernetes pods the following environment variables need to be set to enable notifications.

**env:**

```

- name: ALTIHEX_FPA_TAGGED
  value: "true"
  
- name: ALTIHEX_FPA_REGION
  value: "<region>"

- name: ALTIHEX_FPA_ACCOUNT_ID 
  value: "<account id>"
  
- name: ALTIHEX_FPA_CLUSTER_NAME
  value: "<cluster name>"
  
```

## Scanning

In order to scan application accounts a switch role needs to be configured in each account that you want to scan. This is entered into the scan\_accounts: key in the configuration file. When the cluster process starts up it first forms a cluster, then once the controller node is established it will scan each account by switching roles doing a scan for EC2, ECS and EKS nodes. The result of the scan is fed into a shared configuration and this in turn is broadcast to the other cluster nodes.  There is example code for the security account and an attached node in the repository. **LOCATION HERE**

# Configuration file

The configuration and the connections the proxy will allow through sequentially are as follows:

**allow\_all:**  when set to true it will allow all connections through the proxy. It will log where the connection would be denied of set to false in the fpa-info.log

**connection: direction: outbound:**

​	**allowed: true** This setting will allow all connections through the proxy, it will log them at the point the connection would 		normally fail. This defaults to false. 

​	**transports:** This is a list of allowed transports, if the transport isn’t on this list the proxy will set the connection to **DENIED**. Currently tcp, udp and icmp are supported.

​	**ports:** This is list of allowed transports and ports. The list value must be separated by a ‘/’  character

​	**protocols:** This is a list of allowed TLS versions. Note: The port list must have an entry  of tcp/443.  At present **ECH** is not supported, connections with **ECH** enabled will be blocked on the proxy.

**scan\_accounts:** This is a list of accounts to be scanned for nodes, The first entry on the list is the Arn of the switch role. A space separated parameter that sets region is also required. Each account / region will be scanned using the switch role arn. You can put duplicate accounts with different switch roles and scan using the alternate roles.

**host\_tag\_profiles:** These are the profiles as listed in the node tag / attribute to configure allowed domains. Each host\_tag profile list value will link to a domain\_group: list key

**domain\_groups:** The keys match the host\_tag\_profiles and allow multiple domains to be listed against a host\_tag\_profile. Using ubuntu as an example this allows all the ubuntu package domains to be maintained in one list.

**reverse\_lookup\_address\_cache:** The list keys are used to update the configurations with ip addresses that cannot be verified in the sending packet, at present these are http: or https: connections. The ip addresses are loaded into the configuration and looked up in advance.


## Example Configuration file
```

settings:
  # allow_all: false - default, set to true to allow all connections through the proxy.
  # allow_all: true - allows all traffic
  cluster:
    # quorum_nodes: 1 is the default, set for multiple nodes in intervals of 3
    # notification_ip: 198.51.100.1 is the default, this is a non internet routable bogon address
    # config_file_check_timeout: 300 is the default. This is the check interval, in seconds, for changes to the config.yaml file.
    # node_timeout: 240 – This is a cluster state timeout in seconds. When the timeout expires it will restart.
  session:
    # This is amount of time before the connection table is checked for garbage collection.
    # check_timeout: 30 – Default  in seconds
    # This is the maximum allowed connection table entries - This will trigger session garbage collection
    # max_entries: 1000 – Default in seconds

connection:
  direction:
    outbound:
      allowed: true
      # Allowed transports are tcp, udp and icmp
      transports: 
        - tcp
        - udp
        - icmp
      ports:
        - tcp/80
        - tcp/443
        - udp/53
        - tcp/53
        - udp/123**
      # Allowed TLS protocols
      protocols:
        - TLSv1_3
        - TLSv1_2 

scan_accounts:
	# Arn of switch role account / region
	- arn:aws:iam::<scan account id>:role/altihex-FPA-switch-role
	  eu-west-2
	# Profiles are linked to a specific domain_groups key. 

host_tag_profiles:
	dbserver:
		- ubuntu
		- elastic
		- ntp
    dbclient:
		- ubuntu
		- kibana

# Domain groups are linked to host_tag_profiles above**
domain_groups:
	ubuntu:
		- ubuntu.com
		- eu-west-2.ec2.archive.ubuntu.com
		- security.ubuntu.com**
		- us.archive.ubuntu.com**
		- archive.ubuntu.com**
		- api.snapcraft.io**

	elastic:
		- artifacts.elastic.co

	kibana:
		- artifacts.elastic.co**

	ntp:
		- ntp.ubuntu.com
		- 183.ip-51-89-151.eu

# With some protocols there is no way to verify the domain name used in the domain_groups. This allows them to be looked up in advance and saved in the configuration.
reverse_lookup_address_cache:
	- ntp.ubuntu.com
	- 183.ip-51-89-151.eu
	
	
```
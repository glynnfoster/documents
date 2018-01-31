One of the really great features of Oracle Solaris 11.4 is now the ability to deploy more complex network configuration through Automated Installer (AI) using System Configuration profiles. A huge amount of work has been done to move all network configuration into the Service Management Facility (SMF) configuration repository. This now means that administrators can configure datalinks and IP interfaces out of the box, something that used to be done through first boot services or other scripting.

There are two SMF services that are responsible for managing this - `svc:/network/datalink-management` and `svc:/network/ip-interface-management`. The Oracle Solaris 11.4 documentation has the comprehensive list of SMF property groups and properties that can be used to configure a wide set of objects, along with some examples that can be found in `/usr/share/auto_install/sc_profiles/`, but here's some examples to get started.

#### Single IP over AI booted datalink

In this example we are going to configure the datalink that we have network booted over (typically net0) with an IPv4 and IPv6 address. The IPv4 address will use a set of AI client side variables (as defined by {{AI_*}}) that we've either picked up through DHCP, or defined in OBP. This will also provide a global static route. You will see here that we only provide configuration for the `svc:/network/ip-interface-management:default`service instance, since by default we'll have our datalinks pre-configured using standard datalink vanity naming (i.e., `net0`, `net1`, etc.).

```
  <service name="network/ip-interface-management" version="1" type="service">
    <instance name="default" enabled="true">
      <property_group name="interfaces" type="application">
        <!-- Interface configuration for device that was network booted, typically net0 -->
        <property_group name="{{AI_NETLINK_VANITY}}" type="interface-ip">
          <property name="address-family" type="astring">
            <astring_list>
              <value_node value="ipv4"/>
              <value_node value="ipv6"/>
            </astring_list>
          </property>
          <!-- IPv4 static address that we pickup from AI client side variables -->
          <property_group name="v4" type="address-static">
            <propval name="ipv4-address" type="astring" value="{{AI_IPV4}}"/>
            <propval name="prefixlen" type="count" value="{{AI_IPV4_PREFIXLEN}}"/>
            <propval name="up" type="astring" value="yes"/>
          </property_group>
          <!-- IPv6 addrconf address -->
          <property_group name="v6" type="address-addrconf">
            <propval name="interface-id" type="astring" value="::"/>
            <propval name="prefixlen" type="count" value="0"/>
            <propval name="stateful" type="astring" value="yes"/>
            <propval name="stateless" type="astring" value="yes"/>
          </property_group>
        </property_group>
      </property_group>
      <!-- Global static route, also picked up from AI client side variable  -->
      <property_group name="static-routes" type="application">
        <property_group name="route-1" type="static-route">
          <propval name="destination" type="astring" value="default"/>
          <propval name="family" type="astring" value="inet"/>
          <propval name="gateway" type="astring" value="{{AI_ROUTER}}"/>
          <propval name="isgateway" type="boolean" value="true"/>
        </property_group>
      </property_group>
    </instance>
  </service>
```

#### An aggregation over `net0` and `net1`

In this example we're creating a datalink aggregation over `net0` and `net1` using Datalink Multipathing (dlmp), so you'll see that we have configured properties in the `svc:/network/datalink-management:default` and `svc:/network/ip-interface-management:default` service instances. In this case we're statically defining an IPv4 address with associated auto-configured IPv6 address. In this case we're not using an AI client side variables in the configuration.

```
  <service name="network/datalink-management" version="1" type="service">
    <instance name="default" enabled="true">
      <property_group name="datalinks" type="application">
        <!-- aggregation of net0 and net1 -->
        <property_group name="aggr1" type="datalink-aggr">
          <propval name="aggr-mode" type="astring" value="dlmp"/>
          <propval name="force" type="boolean" value="false"/>
          <propval name="key" type="count" value="0"/>
          <propval name="media" type="astring" value="Ethernet"/>
          <propval name="num-ports" type="count" value="2"/>
          <property name="ports" type="astring">
            <astring_list>
              <value_node value="net0"/>
              <value_node value="net1"/>
            </astring_list>
          </property>
        </property_group>
      </property_group>
    </instance>
  </service>
  <service name="network/ip-interface-management" version="1" type="service">
    <instance name="default" enabled="true">
      <property_group name="interfaces" type="application">
        <!-- aggr1 interface configuration -->
        <property_group name="aggr1" type="interface-ip">
          <property name="address-family" type="astring">
            <astring_list>
              <value_node value="ipv4"/>
              <value_node value="ipv6"/>
            </astring_list>
          </property>
          <!-- IPv4 static address -->
          <property_group name="v4" type="address-static">
            <propval name="ipv4-address" type="astring" value="10.10.210.80"/>
            <propval name="prefixlen" type="count" value="24"/>
            <propval name="up" type="astring" value="yes"/>
          </property_group>
          <!-- IPv6 addrconf address -->
          <property_group name="v6" type="address-addrconf">
            <propval name="interface-id" type="astring" value="::"/>
            <propval name="prefixlen" type="count" value="0"/>
            <propval name="stateful" type="astring" value="yes"/>
            <propval name="stateless" type="astring" value="yes"/>
          </property_group>
        </property_group>
      </property_group>
      <!-- Global static route -->
      <property_group name="static-routes" type="application">
        <property_group name="route-1" type="static-route">
          <propval name="destination" type="astring" value="default"/>
          <propval name="family" type="astring" value="inet"/>
          <propval name="gateway" type="astring" value="10.10.210.1"/>
          <propval name="isgateway" type="boolean" value="true"/>
        </property_group>
      </property_group>
    </instance>
  </service>
```

#### An IPMP interface over `net0` and `net1`

In this example we're creating an IP Network Multipathing (IPMP) interface over `net0` and `net1`. Because this is at the IP layer, we're only configuring properties in the `svc:/network/ip-interface-management:default`service instance. Also, in this case we're using an interface specific route, different to the global static routes we had defined in the previous example.

```
  <service name="network/ip-interface-management" version="1" type="service">
    <instance name="default" enabled="true">
      <property_group name="interfaces" type="application">
        <!-- net0 interface configuration -->
        <property_group name="net0" type="interface-ip">
          <property name="address-family" type="astring">
            <astring_list>
              <value_node value="ipv4"/>
              <value_node value="ipv6"/>
            </astring_list>
          </property>
          <propval name="ipmp-interface" type="astring" value="ipmp1"/>
        </property_group>
        <!-- net1 standby interface configuration -->
        <property_group name="net1" type="interface-ip">
          <property name="address-family" type="astring">
            <astring_list>
              <value_node value="ipv4"/>
              <value_node value="ipv6"/>
            </astring_list>
          </property>
          <propval name="ipmp-interface" type="astring" value="ipmp1"/>
          <property_group name="ip" type="interface-protocol-ip">
            <propval name="standby" type="astring" value="on"/>
          </property_group>
        </property_group>
        <!-- IPMP interface configuration -->
        <property_group name="ipmp1" type="interface-ipmp">
          <property name="address-family" type="astring">
            <astring_list>
              <value_node value="ipv4"/>
              <value_node value="ipv6"/>
           </astring_list>
          </property>
          <property name="under-interfaces" type="astring">
            <astring_list>
              <value_node value="net0"/>
              <value_node value="net1"/>
            </astring_list>
          </property>
          <!-- IPv4 static address, picked up from AI client side variables -->
          <property_group name="v4" type="address-static">
            <propval name="ipv4-address" type="astring" value="10.10.210.80"/>
            <propval name="prefixlen" type="count" value="22"/>
            <propval name="up" type="astring" value="yes"/>
          </property_group>
          <!-- IPv6 addrconf address -->
          <property_group name="v6" type="address-addrconf">
            <propval name="interface-id" type="astring" value="::"/>
            <propval name="prefixlen" type="count" value="0"/>
            <propval name="stateful" type="astring" value="yes"/>
            <propval name="stateless" type="astring" value="yes"/>
          </property_group>
          <!-- Per interface route -->
          <property_group name="static-routes" type="application">
            <property_group name="route-1" type="static-route">
              <propval name="destination" type="astring" value="default"/>
              <propval name="family" type="astring" value="inet"/>
              <propval name="gateway" type="astring" value="10.10.120.1"/>
              <propval name="isgateway" type="boolean" value="true"/>
            </property_group>
          </property_group>
        </property_group>
      </property_group>
    </instance>
  </service>
```

#### Virtual NIC with VLAN ID over `net0`

In this example we're going to create a VNIC over `net0` with VLAN ID 137, and plumb an IP on this VNIC using DHCP. In this case we'll configure both the `svc:/network/datalink-management:default` and `svc:/network/ip-interface-management:default` service instances - one to define the VNIC, and the other to define the DHCP IP.

```
    <service name="network/datalink-management" version="1" type="service">
      <instance name="default" enabled="true">
        <property_group name="datalinks" type="application">
          <!-- vnic configured over net0 -->
          <property_group name="vnic0" type="datalink-vnic">
            <propval name="linkover" type="astring" value="net0"/>
            <propval name="mac-address-type" type="astring" value="random"/>
            <propval name="media" type="astring" value="Ethernet"/>
            <propval name="vid" type="astring" value="137"/>
          </property_group>
        </property_group>
      </instance>
    </service>
    <service name="network/ip-interface-management" version="1" type="service">
      <instance name="default" enabled="true">
        <property_group name="interfaces" type="application">
          <!-- configuration for vnic0 -->
          <property_group name="vnic0" type="interface-ip">
            <property name="address-family" type="astring">
              <astring_list>
                <value_node value="ipv4"/>
                <value_node value="ipv6"/>
              </astring_list>
            </property>
            <!-- IPv4 DHCP address -->
            <property_group name="v4" type="address-dhcp"/>
            <!-- IPv6 addrconf address -->
            <property_group name="v6" type="address-addrconf">
              <propval name="interface-id" type="astring" value="::"/>
              <propval name="prefixlen" type="count" value="0"/>
              <propval name="stateful" type="astring" value="yes"/>
              <propval name="stateless" type="astring" value="yes"/>
            </property_group>
          </property_group>
        </property_group>
      </instance>
    </service>
```

The above SMF profile examples are just snippets as part of the overall SC profile that gets applied to the AI service. Many other additional configurations are possible, including defining datalink properties, flows and other network objects such as bridges, VLANs and VXLANs. To apply a profile to an existing AI service, you may either need to create the profile, or update the profile (if you have it already created). It's a simple as the following:

```
# installadm create-profile -n default-sparc -p my_profile -f /path/to/sc_profile.xml
```

or

```
# installadm update-profile -n default-sparc -p my_profile -f /path/to/updated_sc_profile.xml
```

So now's the time to move to Oracle Solaris 11.4 and remove some of your post-install customizations. The combination of AI and SMF profiles provides a great way to get a reliable and re-producible system configuration out of the box - crucially important when you're installing 1000's of systems in your cloud environment.
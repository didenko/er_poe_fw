EdgeMax firewall study
======================

There are a few templates on the Internet for configuring firewall rules on Ubiquiti _EdgeRouter_ but no from-scratch guide which may be preferred for better understanding. Also, for visual people at least some imagery may be helpful.

As usual, there are many way to skin a cat. Although there are good practices for configuring firewall rules, there is not a best one. Below is _a way_, to approach the task, without any claims of it's quality or fitness.

While presenting configuration examples in this write-up, the configuration code is simplified to capture the concepts. In firewall riles `disabled` filters are not shown, unless relevant for understanding. Other lines not relevant in a paragraph context are omitted as well. Finally, empty bracket sequences are also dropped.

Interfaces
----------

This write-up walks through a SOHO firewall rules configuration reasoning. It assumes a SOHO setup on _EdgeRouter POE_ with three networks: _LAN_, _WAN_, and _DMZ_. The _LAN_ network is on the single Ethernet connection on `eth0` port of the router. The _WAN_ network is essentially an Ethernet link up to an ISP port via `eth1` port. There are not more than three devices on _DMZ_ in this example, so we will use the build-in switch for that purpose. The `eth2`, `eth3`, and `eth4` ports are assigned to `switch0` interface and all firewall rules then consider only `switch0` as _DMZ_ network interface.

Here is the relevant extract from the configuration:

```
interfaces {
    ethernet eth0 {
        description LAN
    }
    ethernet eth1 {
        description WAN
    }
    ethernet eth2
    ethernet eth3
    ethernet eth4
    switch switch0 {
        description DMZ
        switch-port {
            interface eth2
            interface eth3
            interface eth4
        }
    }
}
```

Canvas
------

First the configuration will be presented in visual form on a canvas, like this:

![Canvas](./0_canvas.png)

The main area is a four-sided field, where three sides dedicated to _LAN_, _DMZ_, and _WAN_ interfaces, as marked. The fourth, right, side does not correspond to an interface, but rather to the `local` traffic destination, designated for the router management.

Each canvas side is divided in two parts, which correspond to `in` and `out` traffic directions. Traffic always flows from an `in` to an `out` on the canvas, including the imaginary `local` "`out`". If line does not extend deep into canvas on an `in` side, it means "to any destination". Likewise if a line does not extend deep into canvas on an `out` side, it means "from any source".

The type and handling of traffic is shown with colors and shapes of lines and badges in the legend below the canvas above.

Local network
-------------

The _LAN_ network is the most permissive for traffic flowing out of it, yet it also needs to have some safety guards for traffic coming from other directions. Valid incoming traffic  (the `in` direction) is allowed to flow to any other destination, while only valid established connections' traffic is allowed to leave from the router into the _LAN_ network (the `out` direction).

![LAN](./1_lan.png)

The configuration excerpt demonstrates the _LAN_-related rule sets:

```
firewall {
    name LAN_IN {
        default-action drop
        description "incoming on LAN"
        rule 1 {
            action drop
            description "LAN invalid"
            state {
                invalid enable
            }
        }
        rule 2 {
            action accept
            description "LAN all valid"
            state {
                established enable
                new enable
                related enable
            }
        }
    }
    name LAN_OUT {
        default-action drop
        description "LAN outcoming"
        rule 1 {
            action drop
            description "LAN new & invalid"
            state {
                invalid enable
                new enable
            }
        }
        rule 2 {
            action accept
            description "LAN valid existing"
            state {
                established enable
                related enable
            }
        }
    }
...
```

You may notice the similar rule pattern in configuration examples below: first come rules which drop as much traffic as reasonable to avoid unnecessary processing. Then come permissive rules, which accept the permitted traffic. Finally, the rule set's `default-action` clause set to `drop` which discards all unrecognized traffic as a safety precaution.

In addition to configuring the rule sets we also need to bind them to interface configuration, like this:

```
    ethernet eth0 {
        description LAN
        firewall {
            in {
                name LAN_IN
            }
            out {
                name LAN_OUT
            }
        }
    }
```

```
firewall {
    name DMZ_IN {
        default-action drop
        description "incoming on DMZ"
        rule 1 {
            action drop
            description "DMZ invalid"
            state {
                invalid enable
            }
        }
        rule 2 {
            action accept
            description "DMZ new to WAN"
            state {
                new enable
            }
        }
        rule 3 {
            action accept
            description "DMZ valid established"
            state {
                established enable
                related enable
            }
        }
    }
    name DMZ_LOCAL {
        default-action drop
        description "DMZ to router"
    }
    name LAN_IN {
        default-action drop
        description "incoming on LAN"
        rule 1 {
            action drop
            description "LAN invalid"
            state {
                invalid enable
            }
        }
        rule 2 {
            action accept
            description "LAN all valid"
            state {
                established enable
                new enable
                related enable
            }
        }
    }
    name LAN_OUT {
        default-action drop
        description "LAN outcoming"
        rule 1 {
            action drop
            description "LAN new & invalid"
            state {
                invalid enable
                new enable
            }
        }
        rule 2 {
            action accept
            description "LAN valid existing"
            state {
                established enable
                related enable
            }
        }
    }
    name WAN_IN {
        default-action drop
        description "incoming on WAN"
        rule 1 {
            action drop
            description "WAN new & invalid"
            state {
                invalid enable
                new enable
            }
        }
        rule 2 {
            action accept
            description "WAN valid established"
            state {
                established enable
                related enable
            }
        }
    }
    name WAN_LOCAL {
        default-action drop
        description "WAN to router"
    }
}
interfaces {
    ethernet eth0 {
        description LAN
        firewall {
            in {
                name LAN_IN
            }
            out {
                name LAN_OUT
            }
        }
    }
    ethernet eth1 {
        description WAN
        firewall {
            in {
                name WAN_IN
            }
            local {
                name WAN_LOCAL
            }
        }
    }
    ethernet eth2
    ethernet eth3
    ethernet eth4
    switch switch0 {
        description DMZ
        firewall {
            in {
                name DMZ_IN
            }
            local {
                name DMZ_LOCAL
            }
        }
        switch-port {
            interface eth2
            interface eth3
            interface eth4
        }
    }
}
```
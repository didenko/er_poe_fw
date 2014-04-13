EdgeMax firewall study
======================

There are a few templates on the Internet for configuring firewall rules on Ubiquiti _EdgeRouter_ but no from-scratch guide which may be prefered for better understanding. Also, for visual people at least some imagery may be helpful.

This write-up walk through a SOHO firewall rules configuration reasoning. It assumes a SOHO setup on _EdgeRouter POE_ with three networks: _LAN_, _WAN_, and _DMZ_. The _LAN_ network is on the single Ethernet connection on `eth0` port of the router. The _WAN_ network is essentially an ethernnet link up to an ISP port via `eth1` port. There are not more than three devices on _DMZ_ in this example, so we will use the build-in switch for that purpose. The `eth2`, `eth3`, and `eth4` ports are assigned to `switch0` interface and all firewall rules then consider only `switch0` as _DMZ_ network interface.

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
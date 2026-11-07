---
icon: wheelchair
---

# Accessing Windows

#### **Remote Desktop Protocol (RDP)**

* RDP uses a client/server architecture
* Important to note that RDP listens by default on logical port `3389`

> In simpler terms, we could consider a network subnet a street in a town (the corporate network), an IP address in that subnet assigned to a host as a house on that street, and logical ports as windows/doors that can be used to access the house.

* We can look for saved Remote Desktop Files (`.rdp`) while on an engagement.

> It typically contains the remote computer's IP address or hostname, screen resolution preferences and username

#### **Using xfreerdp**

To connect to a Windows machine via RDP, we can use **xfreerdp.** The command to the RDP is listed below

```jsx
xfreerdp /v:<targetIp> /u:<username> /p:<password>
```

Other RDP clients exist, such as [Remmina](https://remmina.org/) and [rdesktop](http://www.rdesktop.org/)

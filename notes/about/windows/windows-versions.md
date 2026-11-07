---
icon: universal-access
---

# Windows Versions

#### **1. To find information about the operating system**

```jsx
Get-WmiObject
```

This cmdlet can be used to get instances of WMI classes or information about available WMI classes

#### **2. To find the version and build number of our system**

We can easily obtain this information using the `win32_OperatingSystem` class

```jsx
Get-WmiObject -Class win32_OperatingSystem | select Version,BuildNumber
```

#### 3. To get a process listing

```jsx
Get-WmiObject -Class Win32_Process
```

#### 4. To get a listing of services

```jsx
Get-WmiObject -Class Win32_Service
```

#### 5. To get BIOS information

```jsx
Get-WmiObject -Class Win32_Bios
```

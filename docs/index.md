# Welcome to Vascular Platform

Thank you for choosing Vascular. This guide will walk you through registration, setup, and integration.

---

## 1. Register

To get started, please [register an account](https://yourdomain.com/register).

---

## 2. Sign In

If you already have an account, [sign in here](https://yourdomain.com/signin).

---

## 3. Running Vascular

Before running Vascular, ensure you meet the following requirements.

### 3.1 Prerequisites

Before you begin, make sure you have:

- An active subscription  
- Access credentials for our private container registry  
  *(Vascular is distributed as a container image.)*  
- A valid license file (`license.key`)  
  You can download this from your Customer Dashboard after registration and payment

---

### 3.2 Pull the Image from the Registry

Use the following command to pull the latest version of the Vascular container:

```bash
docker pull vascular.registry.com/inbox:latest
```


### 3.3 Activating the Application

The application requires the license file to start.
You can provide the license file in two ways:

#### 1. Default location
Mount the license file inside the container at:


```bash
/etc/vascular-inbox/license.key
```

#### 2. Custom location
Mount the license file anywhere you prefer and provide the path using:

```bash
--license-file=/custom/path/license.key
```

If the license file is missing or invalid, the application will not start.

** Example: Kubernetes Deployment with Custom License File Path **

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vascular-inbox
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vascular-inbox
  template:
    metadata:
      labels:
        app: vascular-inbox
    spec:
      containers:
        - name: vascular-inbox
          image: vascular.registry.com/inbox:latest
          args: ["--license-file=/etc/vascular-inbox/license.key"] # optional if using default
          volumeMounts:
            - name: license
              mountPath: /etc/vascular-inbox/license.key
              subPath: license.key
              readOnly: true
      volumes:
        - name: license
          secret:
            secretName: vascular-license

```
## 4. Client Integration

You can integrate Vascular with your client applications using one of our open-source plugins.

### 4.1 Download Client Plugins

Web (React)
https://github.com/vascular-io/vascular-js

React Native
https://github.com/vascular-io/vascular-js

Android (Java/Kotlin)
https://github.com/vascular-io/vascular-kotlin

iOS (Swift)
https://github.com/vascular-io/vascular-swift

Flutter
https://github.com/vascular-io/vascular-flutter


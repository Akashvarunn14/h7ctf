# H7CTF - b2r Writeup

This repository contains a detailed, step-by-step writeup for the H7Corp (b2r) machine from H7CTF.

## 📝 The Writeup

The full writeup can be found in the file:
* [**H7CTF- b2r writeup.md**](H7CTF-%20b2r%20writeup.md)

This writeup covers the entire process, from initial enumeration to rooting the machine, including:
* **Information Disclosure** via an exposed `.git` directory
* **gRPC Enumeration** & Reflection Bypass
* **Broken Access Control (BAC)** to gain admin privileges
* **Insecure Deserialization (RCE)** using Python `pickle`
* **Privilege Escalation** via a SUID command injection
* **Pivoting** to a privileged `root` gRPC service

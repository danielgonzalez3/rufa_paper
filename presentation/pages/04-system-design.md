---
layout: section
---

# System Design

AWS architecture · Three-tier deployment · Data flow

---

# AWS Architecture Overview

<div class="flex justify-center mt-4">
<img src="../public/rufa_system.png" class="max-h-[35vh] max-w-full rounded shadow object-contain" />
</div>

<!--
- The VPC boundary is important here.
- The RDS instance sits in a private subnet, which means it has no public IP and cannot be reached from the internet directly.
- Only the EC2 instances in the application tier can connect to it, through a security group rule.
- This limits the attack surface significantly compared to a publicly accessible database.
- The two availability zones for the EC2 tier provide fault tolerance: if one availability zone goes down due to infrastructure failure, the load balancer continues routing to the other zone.
- CloudFront serves both the single-page application static assets and proxies API calls from the browser to the backend, which reduces the load on origin servers and provides geographic caching.
-->


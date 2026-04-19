# AWS Assignment 4 – Serverless API with Security, Custom Domain and WAF

---

## Overview

This project demonstrates the design and implementation of a fully serverless API on AWS.

The goal was not just to make it work, but to build it in a way that reflects how a real system would be structured:
- clear separation of components
- controlled access
- external exposure through a custom domain
- protection at the edge

The architecture uses:
- AWS Lambda for compute
- API Gateway for routing and exposure
- DynamoDB for storage
- IAM for least-privilege access
- API Keys and Usage Plans for access control
- AWS WAF for rate limiting
- ACM for TLS
- Cloudflare for DNS

---

## Objective

The aim of this assignment was to move beyond basic AWS services and start thinking in terms of system design and control.

Specifically:
- build a working serverless API
- connect multiple AWS services together
- control who can access the API and how
- expose the API externally in a realistic way
- introduce protection mechanisms such as rate limiting
- validate everything from outside AWS

This is where things started to feel less like individual services and more like a system.

---

## Architecture

Client → Cloudflare DNS → API Gateway Custom Domain → API Gateway Routes → Lambda → DynamoDB  
                                                        ↓  
                                                      AWS WAF

(Architecture diagram to be added)

---

## 1. DynamoDB

A `students` table was created to store data submitted via the API.

Design decisions:
- Partition key: `id`
- On-demand capacity
- Simple structure to focus on integration rather than schema complexity

Each record contains:
- id
- timestamp
- payload

![DynamoDB Table](./images/dynamodb-table.png)  
![DynamoDB Items](./images/dynamodb-items.png)

---

## 2. Lambda Functions

Two Lambda functions were created to handle the API logic.

### submitStudent (POST)
- receives JSON input
- generates a UUID
- adds a timestamp
- writes the item to DynamoDB

### getStudents (GET)
- scans the DynamoDB table
- returns all stored items

Each function was kept focused on a single responsibility.

![Lambda Overview](./images/lambda-overview.png)  
![GET Students Lambda](./images/lambda-getstudents.png)

---

## 3. IAM – Least Privilege

Permissions were scoped down to only what was required:

- PutItem for the POST function
- Scan for the GET function

This reinforces the idea of limiting blast radius and building secure defaults.

![PutItem Policy](./images/iam-putitem-policy.png)  
![Scan Policy](./images/iam-scan-policy.png)

---

## 4. API Gateway

API Gateway was used to expose the Lambda functions.

Endpoints:
- POST /submit
- GET /students

Configured using Lambda proxy integration to keep the flow simple.

![POST Method](./images/api-post-method.png)  
![GET Method](./images/api-get-method.png)  
![API Stage](./images/api-stage.png)

---

## 5. Testing Inside AWS

Both endpoints were tested within API Gateway before exposing externally.

This allowed validation of:
- request structure
- response format
- Lambda execution

![POST Test](./images/api-post-test.png)  
![GET Test](./images/api-get-test.png)

---

## 6. API Key and Usage Plan

Access to the GET /students endpoint was restricted using:
- API key
- usage plan
- stage association

This introduced controlled access to the API.

![API Key](./images/api-key.png)  
![Usage Plan](./images/usage-plan.png)

---

## 7. External Testing from VPS

The API was tested externally using curl from a VPS.

### Without API key

    curl https://api.pietrodevops.uk/students

Response:  
403 Forbidden

### With API key

    curl -H "x-api-key: YOUR_API_KEY" https://api.pietrodevops.uk/students

Response:  
200 OK with valid JSON data

![No API Key](./images/vps-no-api-key.png)  
![With API Key](./images/vps-with-api-key.png)  
![GET Success](./images/vps-get-success.png)

---

## 8. Custom Domain and DNS

A custom domain was configured to expose the API externally.

Steps:
- ACM certificate created in us-east-1
- API Gateway custom domain configured
- API mapped to stage
- Cloudflare DNS CNAME created

Access via:
https://api.pietrodevops.uk

![ACM Certificate](./images/acm-certificate.png)  
![API Custom Domain](./images/api-custom-domain.png)  
![Cloudflare DNS](./images/cloudflare-dns.png)  
![Custom Domain Test](./images/custom-domain-test.png)

---

## 9. AWS WAF – Rate Limiting

AWS WAF was configured to protect the API.

Setup:
- Web ACL attached to API Gateway
- Rate-based rule
- Limit set to 100 requests per 5 minutes
- Action set to block

![WAF Overview](./images/waf-overview.png)  
![WAF Rule](./images/waf-rule.png)

### Validation

Requests were sent in a loop from the VPS:

    for i in {1..120}; do curl -s -o /dev/null -w "%{http_code}\n" -H "x-api-key: YOUR_API_KEY" https://api.pietrodevops.uk/students; done

- Initial requests returned 200
- After threshold was exceeded, responses returned 403

![Loop 200 Success](./images/vps-loop-200-success.png)  
![Loop 403 Failure](./images/vps-loop-403-failure.png)

---

## Example Requests

### POST

    curl -X POST https://api.pietrodevops.uk/submit \
    -H "Content-Type: application/json" \
    -d '{"name":"Pietro","module":"AWS"}'

### GET without API key

    curl https://api.pietrodevops.uk/students

### GET with API key

    curl -H "x-api-key: YOUR_API_KEY" https://api.pietrodevops.uk/students

---

## Key Learnings

This assignment shifted my thinking from using individual AWS services to designing a system.

Main takeaways:
- how API Gateway, Lambda and DynamoDB connect as a full request flow
- the importance of controlling access using API keys and usage plans
- how to expose a service properly with a custom domain
- how DNS, certificates and API Gateway integrate together
- how to introduce protection using AWS WAF
- how to validate a system externally, not just inside AWS
- how small misconfigurations can break the entire flow

It reinforced that a system is not just about making it work, but making it controlled, observable and safe to expose.

---

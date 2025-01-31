---
layout: post
title: System Design Notes II
subtitle: Notes for system design
# cover-img: /assets/img/path.jpg
thumbnail-img: /assets/img/system_design_pic.jpg
# share-img: /assets/img/path.jpg
tags: [
    System Deign,
]
author: Yuchen Jiang
readtime: true
---

A simple note for system design.

From *System Design Interview Volume 2* by Alex Xu

## Proximity Service

### Basic requirements

- functional
  - Return all businesses based on a user’s location (latitude and longitude pair) and radius.
  - Business owners can add, delete or update a business, but this information doesn’t need to be reflected in real-time.
  - Customers can view detailed information about a business.

- non-functional
  - Highly availability and scalability: able to handle spike in traffic
  - Low latency
  - Data privacy: comply data privacy laws like GDPR and CCPA when designing LBS (Location-Based Service) systems.

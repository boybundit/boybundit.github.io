---
layout: post
title: Multiple Canonical Models
slug: multiple-canonical-models
date: 2025-02-10 20:36:00 +0700
categories: programming
image: /assets/images/2025-02-10-multiple-canonical-models/bounded-contexts.jpg
---

มีอยู่เรื่องหนึ่งที่พบอยู่เรื่อยๆเวลา design microservice หรือ data model (ต่อไปขอเรียกแค่ service ละกัน) ของระบบใหญ่ๆ แล้วมันจะมี entity กลุ่มหนึ่ง ที่มันจะค่อนข้าง hot เช่นพวก customer หรือ product ที่ใครๆก็ต้องใช้ เช่น จะสร้าง sales order ก็ต้องใช้ customer, จะออก invoice ก็ต้องใช้ customer, จะสร้าง customer support ticket ก็ต้องใช้ customer

![image](/assets/images/2025-02-10-multiple-canonical-models/bounded-contexts.jpg)

ไม่รู้ว่าเจ้าพวกนี้มันมีชื่อเรียกจริงๆว่าอะไร ที่เคยเห็นแล้วใกล้เคียงมากที่สุดก็คือ [Canonical Models](https://martinfowler.com/bliki/MultipleCanonicalModels.html) ที่ลุง Martin Fowler แกเรียกไว้ตั้งแต่ปี 2003 แต่เหมือนไม่ค่อยได้ยินคนเรียกกัน 🤔

บ่อยครั้งเราจะเห็นเจ้าพวกนี้ถูก design ไปในทางที่ว่า เราต้องมี customer service ตรงกลางอันเดียว จะได้มี single source of truth แต่มันจะมีปัญหาอยู่ก็คือ ทุกคนจะมาต้องมา depend กับ service นี้ กลายเป็น single point of failure ไปด้วย แล้ว service นี้เองมันก็จะบวมมากๆ อาจจะมีเป็นร้อยเป็นพัน fields เลยที่มี key เป็น customer id โดนจับมารวมตัวกัน เช่น customer อาจจะมี fields
- shipping address
- credit card
- support tier

ยิ่งถ้าบอกว่ามี customer master team คนเดียวเท่านั้นที่จะ update data ตรงนี้ได้ ยิ่งลำบากใหญ่ เพราะจะกลายเป็น proxy เฉยๆ เพราะคนที่รู้จริงๆว่าแต่ละ field ต้องใส่อะไรคือคนที่ใช้ data เหล่านี้ต่างหาก
- shipping address คนที่ใช้จริงๆคือ sales service
- credit card คนที่ใช้จริงๆคือ billing service
- support tier คนที่ใช้จริงๆคือ customer support service

ดังนั้น มันน่าจะดีกว่า ถ้าแต่ละ service จะ own data ของตัวเอง

อีก concept หนึ่งที่ชอบก็คือ มันดูเผินๆเหมือนใครๆก็ต้องใช้ customer แต่จริงๆแล้วอาจจะไม่ใช่ก็ได้นะ
- ใน sales service มันคือ buyer
- ใน billing service มันคือ payer
- ใน support service มันคือ caller 

พวกนี้ล้วนมี properties ของตัวเอง แค่อาจจะมี customer id ร่วมกันจะได้ link กันได้

พอเรียกชื่อให้ต่างกันตาม domain แบบนี้แล้ว มันชัดเจนดีว่าจริงๆมันไม่ใช่ของอย่างเดียวกันขนาดนั้นนี่นา

ลุง Martin แกแนะนำไว้ว่า Single Canonical Model นั้นไม่น่าเวิก เรามาทำ Multiple Canonical Models กันน่าจะดีกว่า

อ่านเพิ่มเติมได้ที่ blog ของลุงแกครับ
- [Bounded Context](https://martinfowler.com/bliki/BoundedContext.html)
- [Multiple Canonical Models](https://martinfowler.com/bliki/MultipleCanonicalModels.html)

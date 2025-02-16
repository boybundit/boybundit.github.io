---
layout: post
title: นับจำนวน unique users ด้วย HyperLogLog
slug: count-unique-users-hyperloglog
date: 2025-02-16 15:24:00 +0700
---

ลองจินตนากาลดูว่า ถ้าเราทำ social media app แล้วอยากโชว์ที่ใต้ viedo ว่ามีจำนวน unique users ที่เข้ามาดูกี่คน เราจะทำยังไง?

ทำแบบง่ายๆได้ด้วยการเก็บ record ของ `user_id` ทั้งหมดที่เคยเข้ามาดูเอาไว้ อาจจะใน relational database ก็ได้ แล้วเราก็แค่ `SELECT DISTINCT COUNT(user_id)` หรือถ้าเราใช้ Redis เราอาจจะเก็บ `user_id` ลง set แทน แล้วเราก็แค่ `SCARD user_set` ก็จะได้จำนวนของใน set แล้ว แต่ว่าข้อเสียของวิธีง่ายๆแบบนี้ก็คือเปลืองที่ เพราะเราต้องจำ `user_id` ทั้งหมด เพื่อเอาไว้เช็คว่ามันซ้ำไหม ทั้งที่ requirement จริงๆ เราแค่อยากรู้จำนวน unique users ไม่ได้อยากรู้ว่ามีใครบ้าง



[HyperLogLog](https://en.wikipedia.org/wiki/HyperLogLog)


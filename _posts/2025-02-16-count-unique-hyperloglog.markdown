---
layout: post
title: นับ unique count ด้วย HyperLogLog
slug: count-unique-hyperloglog
date: 2025-02-16 15:24:00 +0700
---

ลองจินตนากาลดูว่า ถ้าเราทำ social แล้วอยากโชว์จำนวน users ที่เข้ามาดู video กี่คน เราจะทำยังไง?

ทำแบบง่ายๆได้ด้วยการเก็บ record ของ `user_id` ทั้งหมดที่เคยเข้ามาดูเอาไว้ อาจจะใน relational database ก็ได้ แล้วเราก็แค่ `SELECT DISTINCT COUNT(user_id)` เป็นอันเรียบร้อย แต่ข้อเสียคือ มันต้องมานับใหม่ทุกครั้ง

ถ้าเราใช้ Redis เราอาจจะเก็บ `user_id` ลง set แทน เนื่องจาก set เก็บแต่ของที่ไม่ซ้ำกันอยู่แล้ว ดังนั้น จำนสน unique users ก็คือจำนวนของใน set ที่ไม่ต้องนับใหม่ ตอบได้เลยใน O(1) แต่ว่าก็ยังมีข้อเสียคือเปลือง memory เพราะเรายังต้องจำ `user_id` ทั้งหมด เพื่อเอาไว้เช็คว่ามันซ้ำไหม


[HyperLogLog](https://en.wikipedia.org/wiki/HyperLogLog)


---
layout: post
title: ลองเข้ารหัสข้อมูลด้วย tink-go
slug: data-encryption-with-tink-go
date: 2024-05-25 +0700
categories: programming
---

ช่วงนี้ต้องทำงานกับ data ที่ต้อง encrypt/decrypt โดย library ที่ใช้ เรียกใช้งาน [Tink](https://developers.google.com/tink) อีกทีหนึ่ง เลยมาลองเล่นดูว่า ถ้าใช้ [tink-go](https://github.com/tink-crypto/tink-go) ตรงๆต้องทำยังงัยกันบ้าง

โดย use case ของเราคือการ encrypt data และ decrypt กลับมาเป็น plaintext ได้ ซึ่งทั้งคน encrypt และ decrypt เป็น service ของเราเอง จึงสามารถทำได้ง่ายๆด้วย symmetric-key algorithm ที่ shared secret key กัน

แต่ความเสี่ยงของวิธีง่ายๆแบบนี้คือ ถ้า key เกิดหลุดไป hacker ก็จะสามารถ decrypt ข้อมูลได้ทั้งหมดเลย! เค้าเลยมีวิธีทำให้มันปลอดภัยขึ้นอีกขั้น ด้วยการใช้ envelop encryption โดยแทนที่จะมี key อันเดียว เราจะมี 2 อันแทน คือ

1. DEK หรือ Data Encryption Key คือ key ที่เอาไว้ encrypt/decrypt ข้อมูล โดยเราจะไม่เก็บ DEK ไว้ใน database แบบ plaintext แต่จะ encrypt ไว้ด้วย KEK ก่อน
2. KEK หรือ Key Encryption Key คือ key ที่เอาไว้ encrypt/decrypt DEK โดยเราจะไม่เก็บเอาไว้เอง แต่จะไปใส่ไว้ใน Key Management System (KMS) เช่น Vault

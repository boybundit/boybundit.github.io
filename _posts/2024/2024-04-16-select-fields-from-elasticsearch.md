---
layout: post
title: งงไปกับวิธีการเลือก field จาก Elasticsearch ในแบบต่างๆ
slug: select-fields-from-elasticsearch
date: 2024-04-16 +0700
categories: programming
---

ช่วงนี้งานที่ทำอยู่ กำลัง migrate จาก [Elasticsearch](https://www.elastic.co/elasticsearch/) ไปเป็น [OpenSearch](https://opensearch.org/) เลยได้มานั่งอ่าน แบบจริงๆจังๆ 🤓 ซึ่งมีเรื่องที่งงอยู่นานเรื่องนึงก็คือ มันมีหลายวิธีมากในการเลือกเฉพาะ fields ที่ต้องการจากการ search (คล้ายๆกับ select ใน SQL query) โดยมีทั้งหมด 4 วิธีคือ
1. `_source`
2. `fields`
3. `stored_fields`
4. `docvalue_fields`

ซึ่งมันใช้งานได้แทบจะเหมือนกัน คือใส่ list ของ fields ที่ต้องการลงไป แล้วผลลัพธ์ที่ได้ก็เหมือนกันอีกด้วย อ้าว แล้วมีมาทำไมตั้ง 4 แบบล่ะ! 🤯

จะเข้าใจเรื่องนี้ได้ต้องเข้าใจก่อนว่าทั้ง Elasticsearch และ OpenSearch นั้นใช้ [Apache Lucene](https://lucene.apache.org/) เป็น engine หลังบ้านเหมือนกัน ซึ่งหลักการพื้นฐานของ full-text search engine ก็คือมันเป็น [Inverted Index](https://opensearch.org/docs/latest/getting-started/intro/#inverted-index) อธิบายง่ายๆ ก็คือมันจะเก็บไว้ก่อนว่า keyword นี้ มีอยู่ใน document ไหนบ้าง (และมีบ่อยแค่ไหน) ที่นี้พอจะ search ก็ไม่ต้องไปไล่หาทีละ document ละ เราก็ดูเอาจาก index ได้เลย มันเลย search ได้เร็ว

![image](/assets/images/2024-04-16-select-fields-from-elasticsearch/inverted-index.webp "https://spotintelligence.com/2023/10/30/inverted-indexing/")

โดยเจ้า inverted index มันไม่ได้เก็บ original document content ไว้นะ มันเก็บไว้แค่ document id เพราะมันเน้นเอาไว้ search เร็วอย่างเดียว แต่ทีนี้มันก็ไม่ค่อยสะดวก เพราะเวลาเราจะโชว์ search result ปกติเราก็อยากจะโชว์ข้อมูลต่างๆด้วย เช่น search ร้านอาหาร อย่างน้อยก็ควรจะโชว์ ชื่อร้าน รูปของร้าน link ไปที่ web ของร้าน โดย Lucene มันจะเก็บ (store) ข้อมูลพวกนี้ แยกออกมาจากตัว inverted index คิดซะว่ามันเป็น table ใน RDBMS ก็ได้ เพราะมันเก็บแบบ row-based เหมือนกัน โดย primary key คือ document id ทำให้ search ได้ document id ปุ๊บ อยากได้ fields ไหนก็ select ได้ปั๊บ

ข้อสังเกตคือ field แต่ละ field นั้น เราเลือกได้ว่าจะ index หรือเปล่า และจะ store รึเปล่า หรือจะทำทั้งคู่ก็ได้ โดย field ที่เราจะ index คือ field ที่เราจะ search ส่วน field ที่เราจะ store คือ field ที่เราจะใช้หลังจาก search แล้ว เช่น เราต้อง index ชื่อร้าน เพราะเราจะ search และเราอยาก store มันด้วยเพราะต้องใช้ทีหลัง แต่ รูปของร้าน กับ link ไปที่ web ของร้าน เราไม่ต้อง index ก็ได้ แค่ store ไว้ก็พอ

ทีนี้พอมันเก็บ store field แบบ row-based เวลา เราอยากจะ sort หรือ aggregate search result (by column) มันเลยช้า เช่น อยากเรียงผลการค้นหาร้านตามจำนวนดาวที่ได้รีวิว เราอยากจะเก็บ field แบบนี้แบบ [column-oriented](https://en.wikipedia.org/wiki/Column-oriented_DBMS) มากกว่าจะได้เร็ว โดยใน Lucene สามารถเลือกให้เก็บแบบนี้ได้ และเรียกมันว่า `DocValue` (เป็นชื่อที่งงมาก)

![image](/assets/images/2024-04-16-select-fields-from-elasticsearch/row-vs-column-db.jpg "https://dataengineering.wiki/Concepts/Column-oriented+Database")

กลับมาที่ Elasticsearch กัน ปกติเวลาเรา index document ลงไป ซึ่งแน่นอนว่ามันจะไปสร้าง inverted index ไว้สำหรับ search แต่ที่มันทำเพิ่มให้โดย default ก็คือมันจะเก็บ (store) raw JSON document นั้นไว้ให้ด้วย ใน field ที่ชื่อว่า `_source` ดังนั้นเวลาใช้ก็เลยง่าย อยากได้ field ไหนก็ parse เอาจาก `_source` นี่แหละง่ายดี และนี่ก็คือวิธีเลือก field แบบที่ (1)

ส่วนแบบที่ (2) การใช้ `fields` นั้น มันก็ parse เอาจาก `_source` เหมือนกันนี่แหละ แต่เพิ่มเติมคือมันจะ map type ตาม index mapping ให้ด้วย เช่น พวก date ใน raw document อาจจะเป็น text แต่เราสามารถ map ใหม่ตามต้องการได้

ส่วนแบบที่ (3) เกิดจากข้อเสียของการใช้ `_source` field ก็คือมันต้องดึง raw JSON document มาทั้งก้อน แล้วค่อยเลือก field ถ้า document เรามันใหญ่มาก แต่ต้องการเลือกไปโชว์แค่ไม่กี่ field เราอาจจะอยากเก็บ (store) field พวกนั้นแยกต่างหาก โดยถ้าจะทำต้องไป set `"store": true` ใน mapping

ส่วนแบบที่ (4) นั้นไม่ต้อง set อะไรเพราะ Elasticsearch จะเก็บทุก field ลง `DocValue` ให้อัตโนมัติ! 😮 แต่ข้อจำกัดคือ ยกเว้น field ที่เป็น text

Official document ของ Elasticsearch แนะนำให้เราใช้ `fields` เป็นหลักก็เพียงพอแล้ว และใช้ _source ในเคสที่เกิดอยากได้ raw data ส่วน `stored_fields` กับ `docvalue_fields` นั้นเอาไว้ใช้ในเคสที่ต้องการ optimize performance จริงๆจังๆครับ

# Reference
- [Search Fields](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-fields.html)

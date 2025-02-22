---
layout: post
title: ลองเล่น Vector Search ด้วย LangChain
slug: vector-search-langchain
date: 2024-05-01 +0700
categories: data_science
---

วันนี้ไปงาน Microsoft Build: AI Day มา มี demo เรื่องการใช้ vector search น่าสนใจดี เลยกลับมาลองเล่นดูซักหน่อย ซึ่งจริงแล้วๆไม่ใช่ของใหม่ แต่ว่าในตอนนี้ที่ LLM มัน hype มากๆ ไอ้เจ้า vector database มันเลย hype ตามไปด้วย 😂

อธิบายแบบบ้านๆ LLM หรือ technology ที่อยู่เบื้องหลัง ChatGPT มันจะมีอยู่ step หนึ่งคือการที่ต้องแปลง text ให้กลายเป็น vector ก่อน จาก text จะได้กลายเป็นเลขจำนวนจริง ที่ feed เป็น input ให้ ANN ได้ ซึ่ง เจ้า vector ก็หน้าตาเหมือนที่เรียนเลขกันมาเลย ตัวอย่างเช่น เราสามารถแทนตำแหน่งต่างๆในระนาบ 2 มิติ ได้ด้วยจำนวนจริงสองค่า (x, y) เรียกว่าเป็น vector 2 มิติ ทีนี้มันจะขยายไปเป็นกี่มิติก็ได้ เช่น ถ้าเราอาจจะแทนคำว่า "Cat" ด้วย vector (0.12, 0.45, -0,34, 0.82, 0.34) 🐱

ขั้นตอนนี้เค้าเรียกกันว่า embbeding ไอเดียคือเหมือนเราสร้าง features ขึ้นมาใหม่ชุดนึงเลย ที่หวังว่าแต่ละเลขมันอาจจะ represent concept อะไรซักอย่างของ text นั้นได้ โดยวิธีการแปลงมี objective สำคัญก็คือ text ที่ "คล้าย" กัน พอแปลงเป็น vector แล้วก็จะ "คล้าย" กันด้วย วิธีคำนวณว่า vector สองอันมันคล้ายกันกี่มากน้อยก็มีหลากหลายวิธี แบบที่เราคุ้นเคยสมัยเรียนเลข ก็เช่น วัดมุมระหว่าง vector เรียกว่า Cosine Similarity ยิ่งเลขน้อยยิ่งคล้ายกันมาก

![image](/assets/images/2024-05-01-vector-search-langchain/cosine-similarity.png "https://medium.com/@milana.shxanukova15/cosine-distance-and-cosine-similarity-a5da0e4d9ded")

ซึ่งด้วยคุณสมบัตินี้ ทำให้เราสามารถ search คำที่คล้ายกันออกมาได้ เช่น ช่วยหา document ที่เกี่ยวกับ "Cat" ให้หน่อย แต่การจะไปไล่คำนวณ similarilty ของทุก document มันก็จะช้า มันก็จะมีหลาย algorithm มาช่วยให้มันเร็วขึ้น ก็เลยเป็นที่มาของ vector database ที่เข้ามาช่วยทำทั้งหมดที่มาว่าให้เลย แบบ one-stop service เราแค่เก็บ text ลงไป เดี๋ยวจัดการเรื่อง similarity search ให้เองตอน query ไม่ต้องทำอะไรเองเลย ชิลๆ 😎

![image](/assets/images/2024-05-01-vector-search-langchain/vector-store.jpg "https://python.langchain.com/docs/modules/data_connection/vectorstores/")

เอาล่ะ เข้าใจ concept พื้นฐานกันพอแล้ว มาลองเขียนโค้ดกันดีกว่า! โดยเราจะใช้ library ที่ชื่อว่า LangChain โดยเริ่มจาก load file csv ที่มี list ของคำทั้งหมดเข้ามาก่อน

```python
from langchain_community.document_loaders.csv_loader import CSVLoader

loader = CSVLoader(file_path='words.csv', csv_args={
    'fieldnames': ['Words']
})

data = loader.load()
```

```csv
Words
Cat
Dog
Car
Truck
Computer
Laptop
Apple
Orange
Music
Dance
```

หลังจากนั้นเราจะยัดของลง vector store ที่ชื่อ FAISS ของ Facebook โดยขั้นตอนนี้ต้องบอกด้วยว่าจะใช้ model อะไรการสร้าง embeddings ซึ่งเราจะใช้ all-MiniLM-L6-v2 เพราะมัน run local ได้เลย ไม่ต้องยิง API เหมือนพวก model ของ OpenAI

```python
from langchain_community.embeddings import SentenceTransformerEmbeddings
from langchain_community.vectorstores import FAISS

embedding_model = SentenceTransformerEmbeddings(model_name="all-MiniLM-L6-v2")

db = FAISS.from_documents(data, embedding_model)
```

แค่นี้เองเราก็พร้อม search แล้ว โดย function similarity_search นี้จะใช้ L2 distance ในการวัดความคล้ายกันของ vector แล้วคืน top 4 documents ที่คล้ายที่สุดมาให้ (เลขน้อยคือคล้ายมาก)

```python
user_input = "Lion"
results = db.similarity_search_with_score(user_input)

for r in results:
  print(r[0].page_content, '\t', r[1])
```

```sh
Words: Cat       1.2079067
Words: Dog       1.2618062
Words: Apple     1.4635086
Words: Orange    1.4841197
```

จะเห็นว่ามันดูราวกับว่า model มันสามารถเรียนรู้ จน "เข้าใจ" concept อะไรบางอย่าง ว่า Lion มันคล้าย Cat มากกว่า Dog นะ ซึ่งดูมหัศจรรย์มาก 😮 แต่ก่อนถ้าจะทำอะไรแบบนี้ต้องสร้าง knowledge graph แล้วมา expand keywords นู่นนั่นนี่วุ่นวาย

จบแล้ว ง่ายมากเลย ความยากมันไปอยู่ใน model นั่นแหละว่า train มายังงัย 555+

ไปลองเล่นกันดูครับ จะได้ hype กับเค้ารู้เรื่อง 😂

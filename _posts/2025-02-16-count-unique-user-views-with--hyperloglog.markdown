---
layout: post
title: นับจำนวน unique user views ด้วย HyperLogLog
slug: count-unique-user-views-with-hyperloglog
date: 2025-02-16 15:24:00 +0700
---

ลองจินตนากาลดูว่า ถ้าเราทำ social media app แล้วอยากโชว์ที่ใต้ viedo ว่ามีจำนวน unique user views ที่เข้ามาดูกี่คน เราจะทำยังไง?

ทำแบบง่ายๆได้ด้วยการเก็บทุก record ของ `user_id` ทั้งหมดที่เคยเข้ามาดูเอาไว้ (คนหนึ่งอาจจะดูซ้ำได้หลายครั้ง) อาจจะใช้ relational database ก็ได้ แล้วเราก็แค่ `SELECT DISTINCT COUNT(user_id) FROM views` หรือถ้าเราใช้ Redis เราอาจจะเก็บ `user_id` ลง set แทน แล้วเราก็แค่ `SCARD users_set` ก็จะได้จำนวนของใน set แล้ว แต่ว่าข้อเสียของวิธีง่ายๆแบบนี้ก็คือเปลืองที่ เพราะเราต้องจำ `user_id` ทั้งหมด เพื่อเอาไว้เช็คว่ามันซ้ำไหม ทั้งที่ requirement จริงๆ เราแค่อยากรู้จำนวน unique user views ไม่ได้อยากรู้ว่ามีใครบ้าง

ถ้าเราลด requirement ของเราลงมาซํกหน่อย เป็นอยากรู้จำนวน unique user views คร่าวๆ ไม่ต้องเป๊ะก็ได้ แต่ขอให้ใช้ memory น้อยลงหน่อย เราสามารถใช้สิ่งที่เรียกว่า probabilistic data structure เข้ามาช่วยได้

ตัวอย่างเช่น [Linear Counting](https://dl.acm.org/doi/10.1145/78922.78925) วิธีการก็คือ เราจะสร้าง bit array ที่เป็น 0 ทั้งหมดไว้ก่อน แล้วเอา `user_id` เข้า hash function แล้วไปตกช่องไหนก็เปลี่ยนจาก 0 เป็น 1 เอาไว้ แล้วสุดท้ายเราก็แค่นับว่ามี 1 กี่ตัว ก็ได้เป็นจำนวน unique user views แล้ว เพราะตัวที่ซ้ำ พอ hash แล้วมันจะไปตกช่องเดิม วิธีนี้ฉลาดสุดๆไปเลย เพราะเปลืองที่น้อยลงมาก แต่ว่าจะนับผิดได้บ้างถ้าเกิด collision โดยเราสามารถ trade-off ได้ด้วยการ ใช้ bit array ที่ใหญ่หน่อย เช่น ใหญ่กว่าจำนวน users ทั้งหมด

แต่ว่าถ้าจำนวน users เยอะมากๆ ยังมีอีกวิธีที่ประหยัดที่ได้มากสุดๆก็คือ [HyperLogLog](https://en.wikipedia.org/wiki/HyperLogLog) วิธีการก็คือ เราจะจำแค่ว่า จาก `hash(user_id)` ที่เคยเห็นมา เจอ leading zero ยาวสุดกี่ตัว ถ้าผ่านตามาทั้งหมด เจอยาวสุด n ตัว แปลว่าเราต้องไล่ดูมาแล้วอย่างน้อย 2^n คน ฉลาดสุดๆไปเลย!

แต่จะเห็นว่า ยิ่ง n เยอะ range มันจะยิ่งกว้าง ดังนั้นใน algorithm จริงจะมีรายละเอียดมากกว่านี้ เพื่อให้ accurary ของค่าประมาณดีขึ้น หลักการคร่าวๆก็คือ เราจะแบ่ง data ออกเป็น subsets ที่เล็กลง แล้วนับแบบที่ว่า (พอ n เล็กลง error ก็น้อย) เสร็จแล้วค่อยเอาค่าที่ได้ของทุก subsets มารวมกันด้วยวิธีฉลาดๆอีกทีหนึ่ง

ข่าวดีคือ ถ้าใช้ Redis อยู่ มัน implement [probabilistic data structures] มาให้แล้วหลายตัว เราเลือกใช้ได้เลยสบายๆ

```python
import redis
import matplotlib.pyplot as plt

# Connect to Redis
r = redis.Redis(host='localhost', port=6379, db=0)

def test_hll_accuracy(cardinality):
    key = "hll_test"
    r.delete(key)

    for i in range(cardinality):
        r.pfadd(key, 'tesggt' + str(i))

    estimated_cardinality = r.pfcount(key)
    absolute_error = abs(estimated_cardinality - cardinality)
    relative_error = absolute_error / cardinality if cardinality > 0 else 0
    return estimated_cardinality, relative_error

# Test with various cardinalities
cardinalities = [1, 2, 3, 4, 5, 6, 7, 8, 9,
                 10, 20, 30, 40, 50, 60, 70, 80, 90,
                 100, 127, 128, 200, 300, 400, 500, 600, 700, 800, 900,
                 1000, 2000, 3000, 4000, 5000, 6000, 7000, 8000, 9000,
                 10000]#, 50000, 100000, 500000, 1000000]
relative_errors = []
estimated_cardinalities = []

for cardinality in cardinalities:
    estimated, relative_error = test_hll_accuracy(cardinality)
    relative_errors.append(relative_error)
    estimated_cardinalities.append(estimated)

r.delete("hll_test") # Clean up

# Create the plot
plt.figure(figsize=(10, 6))  # Adjust figure size for better visualization
plt.plot(cardinalities, relative_errors, marker='o', linestyle='-')
plt.xscale('log')  # Use logarithmic scale for x-axis to handle wide range of cardinalities
plt.xlabel('Cardinality (Log Scale)')
plt.ylabel('Relative Error')
plt.title('HyperLogLog Relative Error vs. Cardinality')
plt.grid(True)
plt.xticks(cardinalities, [str(c) for c in cardinalities], rotation=45) # Show all cardinality values on x-axis
plt.tight_layout() # Adjust layout to prevent labels from overlapping
plt.show()


# Optional: Plot estimated vs. actual cardinality
plt.figure(figsize=(10, 6))
plt.plot(cardinalities, estimated_cardinalities, marker='o', linestyle='-', label="Estimated")
plt.plot(cardinalities, cardinalities, linestyle='--', label="Actual", color='red') # Plot the actual values for comparison
plt.xscale('log')
plt.xlabel('Cardinality (Log Scale)')
plt.ylabel('Cardinality')
plt.title('HyperLogLog Estimated vs. Actual Cardinality')
plt.grid(True)
plt.xticks(cardinalities, [str(c) for c in cardinalities], rotation=45)
plt.legend()
plt.tight_layout()
plt.show()
```

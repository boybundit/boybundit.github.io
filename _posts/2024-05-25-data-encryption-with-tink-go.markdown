---
layout: post
title: ลองเข้ารหัสข้อมูลด้วย tink-go
slug: data-encryption-with-tink-go
date: 2024-05-25 +0700
categories: programming
---

ช่วงนี้ต้องทำงานกับ data ที่ต้อง encrypt/decrypt โดย library ที่ใช้ เรียกใช้งาน [Tink](https://developers.google.com/tink) อีกทีหนึ่ง เลยมาลองเล่นดูว่า ถ้าใช้ [tink-go](https://github.com/tink-crypto/tink-go) ตรงๆต้องทำยังงัยกันบ้าง

โดย use case ของเราคือการ [encrypt data](https://developers.google.com/tink/encrypt-data) และ decrypt กลับมาเป็น plaintext ได้ ซึ่งทั้งคน encrypt และ decrypt เป็น service ของเราเอง จึงสามารถทำได้ง่ายๆด้วย [symmetric-key algorithm](https://en.wikipedia.org/wiki/Symmetric-key_algorithm) ที่ shared secret key กัน

แต่ความเสี่ยงของวิธีง่ายๆแบบนี้คือ ถ้า key เกิดหลุดไป hacker ก็จะสามารถ decrypt ข้อมูลได้ทั้งหมดเลย! เค้าเลยมีวิธีทำให้มันปลอดภัยขึ้นอีกขั้น ด้วยการใช้ [envelop encryption](https://cloud.google.com/kms/docs/envelope-encryption) โดยแทนที่จะมี key อันเดียว เราจะมี 2 อันแทน คือ

1. DEK หรือ Data Encryption Key คือ key ที่เอาไว้ encrypt/decrypt ข้อมูล โดยเราจะไม่เก็บ DEK ไว้ใน database แบบ plaintext แต่จะ encrypt ไว้ด้วย KEK ก่อน
2. KEK หรือ Key Encryption Key คือ key ที่เอาไว้ encrypt/decrypt DEK โดยเราจะไม่เก็บเอาไว้เอง แต่จะไปใส่ไว้ใน Key Management System (KMS) เช่น [Vault](https://www.vaultproject.io/)

![image](assets/images/2024-05-25-data-encryption-with-tink-go/envelope-encyption.png "https://cloud.google.com/kms/docs/envelope-encryption")

ด้วยวิธีแบบนี้ จะเห็นว่าถ้า database หลุดไปพร้อม DEK hacker ก็ยังไม่สามารถเข้าถึงข้อมูลได้เพราะไม่มี KEK แต่คนอาจจะสงสัยว่า แล้วทำไมไม่เก็บ DEK ใน KMS ไปเลยก็หมดเรื่อง? ที่เราไม่ทำแบบนั้นเพราะท่านี้เราสามารถมี DEK ได้จำนวนมาก จะ generate DEK ใหม่ทุกครั้ง ที่เก็บ data เลยก็ได้ เพราะเราเก็บ DEK ไว้คู่กับ data อยู่แล้ว ท่านี้ก็จะ secure สุดๆไปเลย เพราะทุก record ถูก encrypt ด้วยคนละ key กันหมดเลย hacker ร้องไห้แล้ว 😂

ทีนี้ถ้าจะใช้ key เยอะขนาดนั้นจะไปใช้ KMS มันจะไม่ scalable เพราะ KMS มันเป็น centralized service มีอันเดียว แต่เรามี microservices เยอะแยะไปหมด ด้วยวิธี envelop encryption ทำให้เราใช้ KEK น้อยๆ (เก็บไว้ใน KMS) แต่มี DEK เยอะๆ (เก็บไว้กับ data) ได้ ฉลาดสุดๆไปเลย!

พอเข้าใจพื้นฐานแล้วมาลองเขียนโค้ดกัน โดยเราต้องมี KMS ไว้เก็บ KEK ก่อน เราจะ run vault ด้วย docker-compose ดังนี้

```yaml
services:
  vault:
    image: vault:1.13.3
    restart: always
    container_name: vault
    environment:
      VAULT_DEV_ROOT_TOKEN_ID: "root"
    ports:
      - "8200:8200"
    cap_add:
      - IPC_LOCK
    command: "vault server -dev-tls -dev-listen-address=0.0.0.0:8200"
```

จากนั้นเราจะยิง API ไปสร้าง KEK ด้วย [transit secret engine](https://developer.hashicorp.com/vault/docs/secrets/transit)

```sh
curl --request POST --header "X-Vault-Token: root" \
  --data '{"type":"transit"}' --insecure \
  https://localhost:8200/v1/sys/mounts/transit

curl --request POST --header "X-Vault-Token: root" \
  --data '{"type":"aes128-gcm96"}' --insecure \
  https://localhost:8200/v1/transit/keys/demo-tink
```

โดยความเท่ของ vault transit secret engine ก็คือเราไม่ต้อง expose KEK ออกมาจาก KMS เลย แต่เราสามารถเรียก API เพื่อ generate encrypted DEK และ decrypt DEK ไปใช้ได้เลย

ขั้นแรกเราต้องสร้าง client มาต่อกับ KMS และ KEK ของเราก่อน

```go
const keyURI = "hcvault://localhost:8200/transit/keys/demo-tink"
client, _ := hcvault.NewClient(keyURI, &tls.Config{InsecureSkipVerify: true}, "root")
kekAEAD, _ := client.GetAEAD(keyURI)
newHandle, _ := keyset.NewHandle(aead.AES256GCMKeyTemplate())
```

ต่อไปก็ generate encrypted DEK

```go
keysetAssociatedData := []byte("keyset encryption example")
buf := new(bytes.Buffer)
writer := keyset.NewBinaryWriter(buf)
_ = newHandle.WriteWithAssociatedData(writer, kekAEAD, keysetAssociatedData)
encryptedKeyset := buf.Bytes()
```

แล้วก็ decrypt DEK ก่อนนำไปใช้

```go
reader := keyset.NewBinaryReader(bytes.NewReader(encryptedKeyset))
handle, _ := keyset.ReadWithAssociatedData(reader, kekAEAD, keysetAssociatedData)
```

เท่านี้เราก็มี DEK ที่พร้อม encrypt/decrypt data แล้ว!

```go
primitive, _ := aead.New(handle)

plaintext := []byte("message")
associatedData := []byte("example encryption")
ciphertext, _ := primitive.Encrypt(plaintext, associatedData)

decrypted, _ := primitive.Decrypt(ciphertext, associatedData)
fmt.Println(string(decrypted)) // Output: message
```

สังเกตว่า [tink-go](https://github.com/tink-crypto/tink-go) ออกแบบ API มาดีมาก คือมันจะไม่ expose plaintext DEK ออกมาให้เรา ดังนั้นเราจะไม่เผลอไป save มันเก็บเอาไว้ เพราะที่ถูกต้องคือเราต้อง save encrypted DEK

โค้ดทั้งหมดดูได้จาก <https://github.com/boybundit/demo-tink> ครับ 😄

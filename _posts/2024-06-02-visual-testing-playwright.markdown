---
layout: post
title: Visual Testing ด้วย Playwright
slug: visual-testing-playwright
date: 2024-06-02 +0700
categories: programming
---

เวลาเขียน web frontend test ส่วนที่เป็น การแสดงผล ไม่ใช่ business logic มันมักจะมีปัญหาอยู่ เช่น

1. มันเหมือนเขียน code ซ้ำอีกครั้งนึง เช่น ปุ่มนี้ต้องเป็นคำนี้นะ
2. มันต้องใช้ตาดูอีกทีอยู่ดี เช่น พวก spacing ว่ามันตรงกับ figma จริงๆมั้ย

จริงๆแล้วมันมีวิธี test frontend อีกแบบที่เรียกว่า [Visual Testing](https://docs.cypress.io/app/tooling/visual-testing) หลักการก็ตรงไปตรงมาคือแทนที่เรา assert DOM value ทำไมเราไม่ capture screen หลังจาก render แล้ว assert pixel ไปเลย แบบนี้ก็การันตีได้ว่างานออกมาแบบ pixel perfect! 🎉

โดย [Playwright](https://playwright.dev/docs/test-snapshots) ทำได้ในตัวมันเองอยู่แล้วแบบง่ายมากๆ แบบนี้

```javascript
import { test, expect } from '@playwright/test';

test('visual test', async ({ page }) => {
  await page.goto('https://bundit.net');
  await expect(page).toHaveScreenshot(landing.png);
});
```

ถ้าเรามี figma อยู่แล้ว ก็ export มาใส่ไว้ใน folder

![image](/assets/2024-06-02-visual-testing-playwright/snapshots.png){: .align-center}

หรือถ้ายังไม่มีรูป ก็สั่งให้ capture รูปปัจจุบันไว้ให้ได้โดยสั่ง

```sh
npx playwright test --update-snapshots
```

ที่นี้พอเราแก้โค้ดอะไร แล้วอยาก verify ว่าหน้าตาหลัง render แล้วยัง pixel perfect เหมือนเดิมก็แค่สั่ง

```sh
npx playwright test
```

ถ้าไปแก้อะไรแล้วหน้าตา web ไม่เหมือนเดิม ดู report ได้ง่ายๆเลยว่า diff ตรงไหน

```sh
npx playwright show-report
```

![image](/assets/2024-06-02-visual-testing-playwright/diff.png){: .align-center}

ไปลองใช้กันดูครับ 🚀

<https://github.com/boybundit/demo-visual-test>

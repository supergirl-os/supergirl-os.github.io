---
show: true
width: 4
date: 2022-09-12 00:01:00 +0800
height: 495px
images:
- src: /assets/images/etc/view.jpg
  title: Jiuzhaigou Valley
  # desc: Description 1.
  # link: https://picsum.photos/
- src: /assets/images/etc/basket.png
  title: Basketball
  # desc: Description 2
- src: /assets/images/etc/test.png
- src: /assets/images/etc/zion.jpg
  title: Zion National Park
- src: /assets/images/etc/tahoe.jpg
---

{% include widgets/carousel.html id=page.id images=page.images height=page.height %}

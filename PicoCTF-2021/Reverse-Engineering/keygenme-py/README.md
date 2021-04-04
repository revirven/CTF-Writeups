# keygenme-py
Đây là 1 challenge dạng menu. Tại mục (c) yêu cầu mã kích hoạt bản full của chương trình (cũng chính là flag)
```bash
===============================================
Welcome to the Arcane Calculator, SCHOFIELD!

This is the trial version of Arcane Calculator.
The full version may be purchased in person near
the galactic center of the Milky Way galaxy. 
Available while supplies last!
=====================================================


___Arcane Calculator___

Menu:
(a) Estimate Astral Projection Mana Burn
(b) [LOCKED] Estimate Astral Slingshot Approach Vector
(c) Enter License Key
(d) Exit Arcane Calculator
What would you like to do, SCHOFIELD (a/b/c/d)? 
```
## Source code
Flag gồm 3 phần:
```python
key_part_static1_trial = "picoCTF{1n_7h3_|<3y_of_"
key_part_dynamic1_trial = "xxxxxxxx"
key_part_static2_trial = "}"
key_full_template_trial = key_part_static1_trial + key_part_dynamic1_trial + key_part_static2_trial
```
Ta chỉ cần chú ý đến hàm `check_key` để tìm ra `key_part_dynamic1_trial`
```python
def check_key(key, username_trial):

    global key_full_template_trial

    if len(key) != len(key_full_template_trial):
        return False
    else:
        # Check static base key part --v
        i = 0
        for c in key_part_static1_trial:
            if key[i] != c:
                return False

            i += 1

        # TODO : test performance on toolbox container
        # Check dynamic part --v
        if key[i] != hashlib.sha256(username_trial).hexdigest()[4]: 
            return False
        else:
            i += 1

        if key[i] != hashlib.sha256(username_trial).hexdigest()[5]:
            return False
        else:
            i += 1

        if key[i] != hashlib.sha256(username_trial).hexdigest()[3]:
            return False
        else:
            i += 1

        if key[i] != hashlib.sha256(username_trial).hexdigest()[6]:
            return False
        else:
            i += 1

        if key[i] != hashlib.sha256(username_trial).hexdigest()[2]:
            return False
        else:
            i += 1

        if key[i] != hashlib.sha256(username_trial).hexdigest()[7]:
            return False
        else:
            i += 1

        if key[i] != hashlib.sha256(username_trial).hexdigest()[1]:
            return False
        else:
            i += 1

        if key[i] != hashlib.sha256(username_trial).hexdigest()[8]:
            return False



        return True
```
với `username_trial`:
```python
username_trial = "SCHOFIELD"
```
## Exploit
```python
import hashlib
from cryptography.fernet import Fernet
import base64

username_trial = "SCHOFIELD".encode()

key_part_static1_trial = "picoCTF{1n_7h3_|<3y_of_"
key_part_static2_trial = "}"

key_part_dynamic1_trial = hashlib.sha256(username_trial).hexdigest()[4]
key_part_dynamic1_trial += hashlib.sha256(username_trial).hexdigest()[5]
key_part_dynamic1_trial += hashlib.sha256(username_trial).hexdigest()[3]
key_part_dynamic1_trial += hashlib.sha256(username_trial).hexdigest()[6]
key_part_dynamic1_trial += hashlib.sha256(username_trial).hexdigest()[2]
key_part_dynamic1_trial += hashlib.sha256(username_trial).hexdigest()[7]
key_part_dynamic1_trial += hashlib.sha256(username_trial).hexdigest()[1]
key_part_dynamic1_trial += hashlib.sha256(username_trial).hexdigest()[8]

key_full_template_trial = key_part_static1_trial + key_part_dynamic1_trial + key_part_static2_trial

print(key_full_template_trial)
```
## Output
```bash
revirven@ubuntu:~/Desktop/GitHub/CTF-Writeups/PicoCTF-2021/Reverse-Engineering/keygenme-py$ python3 exploit.py
picoCTF{1n_7h3_|<3y_of_e584b363}

```
## Bonus
Khi nhập mã kích hoạt vào, chương trình sẽ tạo 1 file keygenme.py là bản full của chương trình và mục (b) sẽ được mở khoá
```bash
===============================================
Welcome to the Arcane Calculator, SCHOFIELD!

This is the trial version of Arcane Calculator.
The full version may be purchased in person near
the galactic center of the Milky Way galaxy. 
Available while supplies last!
=====================================================


___Arcane Calculator___

Menu:
(a) Estimate Astral Projection Mana Burn
(b) [LOCKED] Estimate Astral Slingshot Approach Vector
(c) Enter License Key
(d) Exit Arcane Calculator
What would you like to do, SCHOFIELD (a/b/c/d)? c

Enter your license key: picoCTF{1n_7h3_|<3y_of_e584b363}

Full version written to 'keygenme.py'.

Exiting trial version...

===================================================

Welcome to the Arcane Calculator, tron!

===================================================


___Arcane Calculator___

Menu:
(a) Estimate Astral Projection Mana Burn
(b) Estimate Astral Slingshot Approach Vector
(c) Exit Arcane Calculator
What would you like to do, tron (a/b/c)? 
```
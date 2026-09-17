# Jenkins Notes — CI Job aur CD Job Ko Alag Karna

## Topic Kya Hai?

Ab tak jitne bhi projects kiye, usme **ek hi Jenkins job** ke andar do kaam ho rahe the:
- **CI (Continuous Integration)** — code build/compile karna, test karna
- **CD (Continuous Deployment)** — production mein deploy karna

Is video mein seekhenge: **CI ko alag job banayenge, CD ko alag job banayenge.** Jenkins server same rahega, bas job do ho jayengi.

---

## Purana Setup (Diagram Samjho)

```
Developer(s) --> GitHub --> Jenkins --> Ansible --> Production (K8s Cluster)
```

- N number of developers apna code likhte hain
- Code GitHub par push hota hai
- GitHub, Jenkins se integrated hai (webhook)
- Jenkins mein ek hi job thi jisme CI + CD dono the

**CI Part** = Jenkins tak (build + compile + test)
**CD Part** = Ansible tak (production mein deploy)

---

## Problem Kya Thi? (Ek Job Mein Dono Hone Se)

### Jab Sab Sahi Chal Raha Ho
- Developer sahi code likhta hai → commit → push
- Job chalti hai → compile hota hai → perfect output aata hai
- CD part bhi chal jaata hai → production mein deploy ho jaata hai
- Sab smooth chal raha hai, koi problem nahi

### Jab Developer Galti Kare
- Developer ne coding mein mistake kar di
- Commit ho gaya, GitHub mein aaya
- Job chali → compile fail ho gaya ya galat/adhura output aaya
- **Problem yeh hai:** Job ke andar CD part bhi defined tha, isliye wahi **galat output bhi production mein deploy ho jayega**
- Isse **production down/kharab** ho sakta hai

### Real-Life Mein Yeh Bada Risk Hai
- Yahan sirf 3 developers hain, lekin company mein 300 developers ho sakte hain
- Kaunsa developer kab galti karega, pata nahi chalega
- Agar har galti production mein deploy hoti rahi, toh production kabhi stable nahi rahega

---

## Solution — CI Job Alag, CD Job Alag

Industry mein yeh tarika follow kiya jaata hai:
- **CI ka job alag** banao
- **CD ka job alag** banao
- **CD job, CI job par dependent rahega**

**Matlab:** CD job tabhi chalegi jab CI job **successfully pass** ho jaaye (stable mode mein aaye). Agar CI fail ho gayi ya unstable rahi, toh CD chalegi hi nahi — production safe rahega.

> Developer chahe 1000 baar galti kare, koi farak nahi padta — CI job 1000 baar chalegi, output perfect nahi aayega toh CD trigger hi nahi hogi. Jis din output perfect aayega, tabhi CD chalegi.

---

## Practical Steps (Jenkins Mein Kaise Karein)

### Step 1: CI Job Banana
1. Jenkins → **New Item**
2. Naam: `CI`
3. **Freestyle project** select karo
4. Git repository URL daalo (Source Code Management)
5. **Build Triggers** → GitHub hook trigger for GITScm polling (check karo)
6. **Build Steps** add karo (jaisa pehle projects mein kiya tha):
   - Jenkins se rsync command se Dockerfile ko Ansible server par bhejo
   - Ansible par Docker build command chalao (image ka naam automatically `$JOB_NAME` se uthega, jo yahan `CI` hoga)
7. **Apply & Save** — CI job ready

### Step 2: CD Job Banana
1. Jenkins → **New Item**
2. Naam: `CD`
3. **Freestyle project** select karo
4. Description likh sakte ho: "CD"
5. Yahan Git/Source code kuch nahi chahiye
6. **Build Triggers** section mein jao:
   - **"Build after other projects are built"** option check karo
   - Project name mein likho: `CI`
   - Neeche 3 options milenge:
     - Trigger only if build is stable
     - Trigger even if the build is unstable
     - Trigger even if the build is fail
   - **"Trigger only if build is stable"** select karo (hum chahte hain CI perfectly pass ho tabhi CD chale)
7. **Build Steps** mein — Ansible ka playbook run karne ka command do (jo already `/opt/ansible.yml` ya jahan bhi playbook rakha hai)
8. **Apply & Save** — CD job ready

---

## Important Note — Image Name Match Karna

- CI job ka naam `CI` rakha hai, toh build hoke jo image Docker Hub mein jayegi uska naam bhi `CI` hoga (kyunki `$JOB_NAME` variable use hota hai)
- Isliye Kubernetes cluster ke deployment file mein bhi image ka naam `CI` hi likhna padega — warna naam match nahi karega aur deployment fail hogi
- Purane project ka image naam agar deployment file mein likha tha, toh usko change karke `CI` karna padega

---

## Testing Kaise Kiya (Practical Mein Kya Dikha)

### Test 1 — Manual Run
- CI job ko **Build Now** se manually chalaya
- Dekha: CI complete hone ke baad CD **automatically** chal gayi (kuch alag se click nahi kiya)
- Dono jobs successful (pass) ho gayin
- Cluster ka node IP nikal ke browser mein hit kiya → website chal rahi thi

### Test 2 — Sahi Code Push Karke Test
- GitHub mein jaake source code mein kuch change kiya
- Commit kiya
- CI job apne aap trigger hui (webhook se)
- CI pass hui → CD automatically chal gayi
- Website ka content bhi refresh karne par change dikha

### Test 3 — Galat Code Push Karke Test (Sabse Important)
- GitHub mein jaan-bujhke kuch content delete/mismatch kar diya (developer ki galti simulate ki)
- Commit kiya
- CI job chali, lekin **fail ho gayi / unstable mode mein aayi**
- **CD job bilkul nahi chali** — production waisa hi raha, koi change nahi hua
- Isse proof hua ki galat code production tak nahi pahuncha

### Test 4 — Wapas Sahi Code Karke Confirm Karna
- Jo line delete ki thi, wapas paste kar di (galti fix ki)
- Commit kiya
- CI phir se pass hui → CD phir se automatically chali (3rd time)
- Production mein content update ho gaya

---

## Key Takeaways (Yaad Rakhne Wali Baatein)

| Point | Matlab |
|---|---|
| CI aur CD alag jobs | Dono ka kaam clearly separate hai |
| CD, CI par dependent | CD tabhi chalegi jab CI pass ho |
| "Trigger only if stable" | Sabse safe option — sirf successful CI ke baad hi CD chale |
| `$JOB_NAME` variable | Image ka naam job ke naam se automatically set hota hai |
| Fayda | Developer ki galti seedha production tak nahi pahunchti — production hamesha safe/stable rehta hai |

---

## Interview Ke Liye Ek Line Mein

> "Maine CI aur CD process ko do alag Jenkins jobs mein separate kiya. CD job ko CI job par dependent banaya using 'Build after other projects are built' with 'Trigger only if build is stable' condition. Isse yeh fayda hua ki agar developer galat code push kare aur CI fail ho jaye, toh CD trigger hi nahi hoti — production hamesha safe rehta hai, sirf verified aur tested code hi deploy hota hai."

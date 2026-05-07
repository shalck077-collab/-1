# دليل المزامنة السحابية (Cloud Sync Integration Guide)

## نظرة عامة
تم تطوير التطبيق بنموذج **Offline-First** حيث تكون النسخة المحلية هي المرجع الأساسي دائماً. يمكنك الآن ربط التطبيق بأي خدمة سحابية (Firebase, Supabase, API خاص بك، إلخ).

---

## المبادئ الأساسية

### 1. أولوية البيانات المحلية
- **البيانات المحلية دائماً هي المرجع الأساسي** (Offline-First)
- عند التعارض بين البيانات المحلية والسحابية، يتم الاحتفاظ بالبيانات المحلية
- المزامنة السحابية اختيارية وتتم يدوياً أو تلقائياً عند توفر الإنترنت

### 2. آليات المزامنة
- **الرفع (Upload)**: إرسال البيانات المحلية الحالية إلى السحابة
- **التحميل (Download)**: جلب البيانات من السحابة (مع تحذير المستخدم)
- **المزامنة التلقائية**: اختيارية، تعمل فقط عند توفر الإنترنت

---

## خطوات الربط بخدمة سحابية

### الخيار 1: Firebase Realtime Database

#### 1. إعداد Firebase
```javascript
// أضف هذا الكود في بداية script.js بعد التحميل

import { initializeApp } from "https://www.gstatic.com/firebasejs/10.0.0/firebase-app.js";
import { getDatabase, ref, set, get, child } from "https://www.gstatic.com/firebasejs/10.0.0/firebase-database.js";

const firebaseConfig = {
  apiKey: "YOUR_API_KEY",
  authDomain: "YOUR_PROJECT.firebaseapp.com",
  databaseURL: "https://YOUR_PROJECT.firebaseio.com",
  projectId: "YOUR_PROJECT",
  storageBucket: "YOUR_PROJECT.appspot.com",
  messagingSenderId: "YOUR_SENDER_ID",
  appId: "YOUR_APP_ID"
};

const app = initializeApp(firebaseConfig);
const database = getDatabase(app);
```

#### 2. تعديل دالة uploadToCloud
```javascript
async function uploadToCloud() {
    if (!navigator.onLine) {
        playSound('error');
        return alert('لا يوجد اتصال بالإنترنت حالياً');
    }
    
    playSound('click');
    const statusEl = document.getElementById('cloudSyncStatus');
    statusEl.innerHTML = 'جاري الرفع...';
    
    const allData = {
        buildings: loadFromStorage('buildings'),
        workers: loadFromStorage('workers'),
        partnerships: loadFromStorage('partnerships'),
        lastUpdated: new Date().toISOString()
    };

    try {
        // رفع البيانات إلى Firebase
        await set(ref(database, 'users/currentUser/data'), allData);
        
        localStorage.setItem('lastCloudSyncDate', new Date().toISOString());
        statusEl.innerHTML = 'آخر مزامنة (رفع): ' + new Date().toLocaleString('ar-SA');
        playSound('success');
        alert('تم رفع البيانات للسحابة بنجاح');
    } catch (error) {
        console.error("Cloud upload error:", error);
        statusEl.innerHTML = '<span style="color:var(--danger-color)">فشل الرفع</span>';
        playSound('error');
    }
}
```

#### 3. تعديل دالة downloadFromCloud
```javascript
async function downloadFromCloud() {
    if (!navigator.onLine) {
        playSound('error');
        return alert('لا يوجد اتصال بالإنترنت حالياً');
    }

    if (!confirm('تحذير: تحميل البيانات من السحابة سيستبدل البيانات المحلية الحالية. هل تريد الاستمرار؟')) return;

    playSound('click');
    const statusEl = document.getElementById('cloudSyncStatus');
    statusEl.innerHTML = 'جاري التحميل...';

    try {
        // جلب البيانات من Firebase
        const snapshot = await get(child(ref(database), 'users/currentUser/data'));
        
        if (snapshot.exists()) {
            const cloudData = snapshot.val();
            
            // تحديث البيانات المحلية
            saveToStorage('buildings', cloudData.buildings || []);
            saveToStorage('workers', cloudData.workers || []);
            saveToStorage('partnerships', cloudData.partnerships || []);
            
            // إعادة تحميل الجداول
            loadBuildingsTable();
            loadWorkersTable();
            loadPartnershipTable();
            initCharts();
            
            statusEl.innerHTML = 'آخر مزامنة (تحميل): ' + new Date().toLocaleString('ar-SA');
            playSound('success');
            alert('تم تحميل البيانات من السحابة وتحديث النسخة المحلية');
        } else {
            alert('لا توجد بيانات في السحابة');
        }
    } catch (error) {
        console.error("Cloud download error:", error);
        statusEl.innerHTML = '<span style="color:var(--danger-color)">فشل التحميل</span>';
        playSound('error');
    }
}
```

---

### الخيار 2: Supabase (PostgreSQL)

#### 1. إعداد Supabase
```javascript
import { createClient } from 'https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/+esm';

const SUPABASE_URL = 'YOUR_SUPABASE_URL';
const SUPABASE_ANON_KEY = 'YOUR_ANON_KEY';

const supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY);
```

#### 2. تعديل دوال المزامنة
```javascript
async function uploadToCloud() {
    // ... نفس التحقق من الاتصال ...
    
    const allData = {
        buildings: loadFromStorage('buildings'),
        workers: loadFromStorage('workers'),
        partnerships: loadFromStorage('partnerships'),
        lastUpdated: new Date().toISOString()
    };

    try {
        const { data, error } = await supabase
            .from('sync_data')
            .upsert({ id: 1, data: allData }, { onConflict: 'id' });
        
        if (error) throw error;
        
        localStorage.setItem('lastCloudSyncDate', new Date().toISOString());
        statusEl.innerHTML = 'آخر مزامنة (رفع): ' + new Date().toLocaleString('ar-SA');
        playSound('success');
    } catch (error) {
        console.error("Cloud upload error:", error);
        statusEl.innerHTML = '<span style="color:var(--danger-color)">فشل الرفع</span>';
        playSound('error');
    }
}
```

---

### الخيار 3: API خاص بك

#### 1. إعداد API
```javascript
const CLOUD_API_URL = 'https://your-api.com/api/sync';
const API_KEY = 'your_api_key';

async function uploadToCloud() {
    // ... التحقق من الاتصال ...
    
    const allData = {
        buildings: loadFromStorage('buildings'),
        workers: loadFromStorage('workers'),
        partnerships: loadFromStorage('partnerships'),
        lastUpdated: new Date().toISOString()
    };

    try {
        const response = await fetch(`${CLOUD_API_URL}/upload`, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'Authorization': `Bearer ${API_KEY}`
            },
            body: JSON.stringify(allData)
        });

        if (!response.ok) throw new Error('Upload failed');
        
        localStorage.setItem('lastCloudSyncDate', new Date().toISOString());
        statusEl.innerHTML = 'آخر مزامنة (رفع): ' + new Date().toLocaleString('ar-SA');
        playSound('success');
    } catch (error) {
        console.error("Cloud upload error:", error);
        statusEl.innerHTML = '<span style="color:var(--danger-color)">فشل الرفع</span>';
        playSound('error');
    }
}

async function downloadFromCloud() {
    // ... التحقق من الاتصال والتحذير ...
    
    try {
        const response = await fetch(`${CLOUD_API_URL}/download`, {
            headers: {
                'Authorization': `Bearer ${API_KEY}`
            }
        });

        if (!response.ok) throw new Error('Download failed');
        
        const cloudData = await response.json();
        
        // تحديث البيانات المحلية
        saveToStorage('buildings', cloudData.buildings || []);
        saveToStorage('workers', cloudData.workers || []);
        saveToStorage('partnerships', cloudData.partnerships || []);
        
        // إعادة تحميل الجداول
        loadBuildingsTable();
        loadWorkersTable();
        loadPartnershipTable();
        initCharts();
        
        statusEl.innerHTML = 'آخر مزامنة (تحميل): ' + new Date().toLocaleString('ar-SA');
        playSound('success');
    } catch (error) {
        console.error("Cloud download error:", error);
        statusEl.innerHTML = '<span style="color:var(--danger-color)">فشل التحميل</span>';
        playSound('error');
    }
}
```

---

## ميزات الأمان

### 1. مصادقة المستخدم
```javascript
// تخزين معرف المستخدم الفريد
const userId = localStorage.getItem('userId') || generateUniqueId();
localStorage.setItem('userId', userId);

function generateUniqueId() {
    return 'user_' + Date.now() + '_' + Math.random().toString(36).substr(2, 9);
}
```

### 2. تشفير البيانات الحساسة
```javascript
// استخدم مكتبة TweetNaCl.js أو crypto-js
async function encryptData(data, password) {
    // تشفير البيانات قبل الرفع
    const encrypted = CryptoJS.AES.encrypt(JSON.stringify(data), password).toString();
    return encrypted;
}

async function decryptData(encrypted, password) {
    // فك تشفير البيانات بعد التحميل
    const decrypted = CryptoJS.AES.decrypt(encrypted, password).toString(CryptoJS.enc.Utf8);
    return JSON.parse(decrypted);
}
```

---

## اختبار المزامنة

### 1. اختبار محلي
```javascript
// افتح وحدة التحكم (F12) وجرب:
uploadToCloud();  // محاكاة الرفع
downloadFromCloud();  // محاكاة التحميل
```

### 2. اختبار الاتصال
```javascript
// تحقق من حالة الاتصال
console.log(navigator.onLine);  // true أو false

// محاكاة قطع الإنترنت
// في Chrome DevTools: Network > Offline
```

---

## ملاحظات مهمة

✅ **النسخة المحلية دائماً هي المرجع الأساسي**
✅ **المزامنة اختيارية وآمنة**
✅ **التطبيق يعمل بدون إنترنت بكفاءة كاملة**
✅ **يمكنك تغيير خدمة السحابة في أي وقت**

⚠️ **تحذيرات:**
- لا تفقد البيانات المحلية عند الرفع
- تأكد من النسخ الاحتياطية قبل التحميل من السحابة
- استخدم كلمات مرور قوية للمزامنة الآمنة

---

## الدعم والمساعدة

للمزيد من المعلومات عن:
- **Firebase**: https://firebase.google.com/docs
- **Supabase**: https://supabase.com/docs
- **Service Workers**: https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API


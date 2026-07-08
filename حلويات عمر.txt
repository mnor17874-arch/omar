# Create the complete Flask app with SQLite database, chef dashboard, login, and phone number
app_code = r'''from flask import Flask, render_template_string, request, jsonify, redirect, url_for, session, flash
import sqlite3
import random
import os
from datetime import datetime
from functools import wraps

app = Flask(__name__)
app.secret_key = 'halawiyatna_secret_key_2026'

MAX_BUDGET = 150
DB_PATH = 'halawiyatna.db'

# ==================== DATABASE ====================
def init_db():
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    
    # Orders table
    c.execute('''CREATE TABLE IF NOT EXISTS orders (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        customer_name TEXT,
        phone TEXT,
        address TEXT,
        items TEXT,
        total REAL,
        status TEXT DEFAULT 'pending',
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        completed_at TIMESTAMP
    )''')
    
    # Users table
    c.execute('''CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        username TEXT UNIQUE,
        password TEXT,
        name TEXT,
        phone TEXT,
        role TEXT DEFAULT 'customer'
    )''')
    
    # Insert default admin (chef)
    c.execute('''INSERT OR IGNORE INTO users (id, username, password, name, phone, role) 
                 VALUES (1, 'chef', 'chef123', 'الشيف', '01080052877', 'chef')''')
    
    conn.commit()
    conn.close()

init_db()

# ==================== DATA ====================
CONTACT_PHONE = '01080052877'

categories = [
    {"id": "all", "name": "الكل"},
    {"id": "kunafa", "name": "الكنافة"},
    {"id": "basbousa", "name": "البسبوسة"},
    {"id": "oriental", "name": "حلويات شرقية"},
    {"id": "western", "name": "حلويات غربية"},
    {"id": "drinks", "name": "مشروبات"},
]

menu_items = [
    {"id": 1, "name": "كنافة ناعمة بالقشطة", "price": 45, "emoji": "🍯", "category": "kunafa", "desc": "كنافة ناعمة محشية قشطة طازجة"},
    {"id": 2, "name": "كنافة خشنة بالمكسرات", "price": 55, "emoji": "🥜", "category": "kunafa", "desc": "خشنة مقرمشة مع فستق ولوز"},
    {"id": 3, "name": "كنافة بالشوكولاتة", "price": 50, "emoji": "🍫", "category": "kunafa", "desc": "كنافة ناعمة مع شوكولاتة بلجيكية"},
    {"id": 4, "name": "كنافة بالمانجو", "price": 48, "emoji": "🥭", "category": "kunafa", "desc": "كنافة محشية مانجو طازج"},
    {"id": 5, "name": "بسبوسة بالقشطة", "price": 35, "emoji": "🍮", "category": "basbousa", "desc": "بسبوسة طرية مع قشطة غنية"},
    {"id": 6, "name": "بسبوسة بالمكسرات", "price": 40, "emoji": "🌰", "category": "basbousa", "desc": "بسبوسة محشية فستق حلبي"},
    {"id": 7, "name": "بسبوسة بالشوكولاتة", "price": 38, "emoji": "🍫", "category": "basbousa", "desc": "بسبوسة بالشوكولاتة الداكنة"},
    {"id": 8, "name": "بسبوسة بالزبادي", "price": 30, "emoji": "🥛", "category": "basbousa", "desc": "بسبوسة خفيفة بالزبادي الطبيعي"},
    {"id": 9, "name": "بقلاوة بالفستق", "price": 60, "emoji": "🥟", "category": "oriental", "desc": "طبقات رقيقة من البقلاوة بالفستق"},
    {"id": 10, "name": "أم علي", "price": 35, "emoji": "🥣", "category": "oriental", "desc": "أم علي ساخنة بالمكسرات والقشطة"},
    {"id": 11, "name": "مهلبية بالورد", "price": 25, "emoji": "🌹", "category": "oriental", "desc": "مهلبية كريمية بنكهة الورد"},
    {"id": 12, "name": "رز بلبن", "price": 28, "emoji": "🍚", "category": "oriental", "desc": "رز بلبن بالقرفة والمكسرات"},
    {"id": 13, "name": "قطايف بالقشطة", "price": 40, "emoji": "🌙", "category": "oriental", "desc": "قطايف محشية قشطة ومكسرات"},
    {"id": 14, "name": "لقمة القاضي", "price": 30, "emoji": "🍩", "category": "oriental", "desc": "لقمة القاضي مقرمشة بالسكر"},
    {"id": 15, "name": "بلح الشام", "price": 25, "emoji": "🍯", "category": "oriental", "desc": "بلح الشام المقرمش بالعسل"},
    {"id": 16, "name": "عيش السرايا", "price": 45, "emoji": "🍞", "category": "oriental", "desc": "عيش السرايا بالقشطة والمكسرات"},
    {"id": 17, "name": "تشيز كيك", "price": 50, "emoji": "🍰", "category": "western", "desc": "تشيز كيك كريمي بالتوت"},
    {"id": 18, "name": "براوني", "price": 40, "emoji": "🍫", "category": "western", "desc": "براوني شوكولاتة غني"},
    {"id": 19, "name": "كريب بالنوتيلا", "price": 45, "emoji": "🥞", "category": "western", "desc": "كريب فرنسي بالنوتيلا والموز"},
    {"id": 20, "name": "وافل بالآيس كريم", "price": 55, "emoji": "🧇", "category": "western", "desc": "وافل بلجيكي مع آيس كريم فانيليا"},
    {"id": 21, "name": "تيراميسو", "price": 48, "emoji": "☕", "category": "western", "desc": "تيراميسو إيطالي أصيل"},
    {"id": 22, "name": "كب كيك", "price": 35, "emoji": "🧁", "category": "western", "desc": "كب كيك متنوع النكهات"},
    {"id": 23, "name": "دوناتس", "price": 30, "emoji": "🍩", "category": "western", "desc": "دوناتس محشية بالكريمة"},
    {"id": 24, "name": "ماكرون", "price": 55, "emoji": "🍬", "category": "western", "desc": "ماكرون فرنسي متنوع الألوان"},
    {"id": 25, "name": "قهوة تركية", "price": 15, "emoji": "☕", "category": "drinks", "desc": "قهوة تركية أصيلة"},
    {"id": 26, "name": "شاي بالنعناع", "price": 10, "emoji": "🍵", "category": "drinks", "desc": "شاي أخضر بالنعناع الطازج"},
    {"id": 27, "name": "عصير مانجو", "price": 25, "emoji": "🥭", "category": "drinks", "desc": "عصير مانجو طبيعي 100%"},
    {"id": 28, "name": "عصير فراولة", "price": 25, "emoji": "🍓", "category": "drinks", "desc": "عصير فراولة طازج"},
    {"id": 29, "name": "موهيتو", "price": 30, "emoji": "🍋", "category": "drinks", "desc": "موهيتو ليمون ونعناع منعش"},
    {"id": 30, "name": "هوت شوكليت", "price": 30, "emoji": "🍫", "category": "drinks", "desc": "هوت شوكليت كريمي بالمارشميلو"},
]

# ==================== AUTH HELPERS ====================
def login_required(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        if 'user_id' not in session:
            return redirect(url_for('login'))
        return f(*args, **kwargs)
    return decorated

def chef_required(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        if 'user_id' not in session:
            return redirect(url_for('login'))
        conn = sqlite3.connect(DB_PATH)
        c = conn.cursor()
        c.execute("SELECT role FROM users WHERE id = ?", (session['user_id'],))
        result = c.fetchone()
        conn.close()
        if not result or result[0] != 'chef':
            flash('غير مصرح!', 'error')
            return redirect(url_for('index'))
        return f(*args, **kwargs)
    return decorated

# ==================== CUSTOMER PAGES ====================
CUSTOMER_HTML = """
<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>حلوياتك عندنا</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            font-family: 'Segoe UI', Tahoma, sans-serif;
            background: #f5f5f5;
            direction: rtl;
        }
        .header {
            text-align: center;
            padding: 30px 20px;
            background: #2c3e50;
            border-radius: 0 0 20px 20px;
            margin-bottom: 20px;
            color: white;
        }
        .header h1 {
            font-size: 32px;
            color: #f1c40f;
            margin-bottom: 8px;
        }
        .header p { opacity: 0.8; font-size: 15px; }
        .contact-bar {
            background: #c0392b;
            color: white;
            text-align: center;
            padding: 10px;
            font-size: 14px;
            display: flex;
            justify-content: center;
            align-items: center;
            gap: 8px;
            flex-wrap: wrap;
        }
        .contact-bar a {
            color: #f1c40f;
            text-decoration: none;
            font-weight: 700;
        }
        .contact-bar a:hover { text-decoration: underline; }
        .container {
            max-width: 900px;
            margin: 0 auto;
            padding: 0 16px 40px;
        }
        .tabs {
            display: flex;
            gap: 8px;
            overflow-x: auto;
            padding: 4px 0 16px;
            margin-bottom: 8px;
        }
        .tab {
            padding: 10px 20px;
            border-radius: 24px;
            border: 1px solid #ddd;
            background: white;
            cursor: pointer;
            white-space: nowrap;
            font-weight: 600;
            font-size: 14px;
            transition: all 0.2s;
        }
        .tab.active {
            background: #c0392b;
            color: white;
            border-color: #c0392b;
        }
        .menu-grid {
            display: grid;
            grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
            gap: 16px;
            margin-bottom: 20px;
        }
        .item-card {
            border: 1px solid #e0e0e0;
            border-radius: 14px;
            padding: 16px;
            background: white;
            transition: transform 0.15s, box-shadow 0.15s;
        }
        .item-card:hover {
            transform: translateY(-3px);
            box-shadow: 0 6px 20px rgba(0,0,0,0.08);
        }
        .item-emoji { font-size: 36px; display: block; text-align: center; margin-bottom: 8px; }
        .item-name { font-weight: 700; font-size: 15px; text-align: center; margin-bottom: 4px; }
        .item-desc { font-size: 12px; text-align: center; color: #888; margin-bottom: 10px; min-height: 32px; }
        .item-price { text-align: center; font-size: 18px; font-weight: 800; color: #c0392b; margin-bottom: 10px; }
        .add-btn {
            width: 100%; padding: 10px; border: none; border-radius: 10px;
            background: #c0392b; color: white; font-weight: 700; font-size: 14px;
            cursor: pointer; transition: background 0.2s;
        }
        .add-btn:hover { background: #e74c3c; }
        .add-btn:disabled { background: #ccc; cursor: not-allowed; }
        .cart-panel {
            border: 1px solid #e0e0e0; border-radius: 16px; padding: 20px;
            background: white; margin-top: 8px;
        }
        .cart-title { font-size: 18px; font-weight: 700; margin-bottom: 12px; }
        .cart-item {
            display: flex; justify-content: space-between; align-items: center;
            padding: 10px 0; border-bottom: 1px solid #eee;
        }
        .cart-item-name { font-weight: 600; font-size: 14px; }
        .cart-item-price { font-size: 13px; color: #888; }
        .qty-controls { display: flex; align-items: center; gap: 8px; }
        .qty-btn {
            width: 28px; height: 28px; border-radius: 50%;
            border: 1px solid #ddd; background: transparent;
            cursor: pointer; font-weight: 700; display: flex;
            align-items: center; justify-content: center;
        }
        .qty-btn:hover { background: #f0f0f0; }
        .cart-total {
            display: flex; justify-content: space-between; align-items: center;
            padding: 14px 0 6px; font-size: 18px; font-weight: 800;
        }
        .cart-total span:last-child { color: #c0392b; }
        .limit-warning {
            color: #e74c3c; font-size: 12px; text-align: center;
            margin-top: 6px; display: none;
        }
        .limit-warning.visible { display: block; }
        .order-btn {
            width: 100%; padding: 14px; border: none; border-radius: 12px;
            background: #2c3e50; color: white; font-weight: 800;
            font-size: 16px; cursor: pointer; margin-top: 14px;
        }
        .order-btn:hover { background: #34495e; }
        .order-btn:disabled { background: #ccc; cursor: not-allowed; }
        .empty-cart { text-align: center; color: #888; padding: 20px; font-size: 14px; }
        .notification {
            position: fixed; top: 20px; left: 50%; transform: translateX(-50%);
            background: #2c3e50; color: white; padding: 14px 28px;
            border-radius: 12px; font-weight: 600; z-index: 1000;
            opacity: 0; transition: opacity 0.3s; pointer-events: none;
        }
        .notification.show { opacity: 1; }
        .order-confirmation {
            text-align: center; padding: 30px 20px; display: none;
        }
        .order-confirmation.visible { display: block; }
        .big-emoji { font-size: 60px; margin-bottom: 12px; }
        .order-confirmation h2 { color: #27ae60; margin-bottom: 8px; }
        .order-confirmation p { color: #888; margin-bottom: 4px; }
        .chef-status {
            display: flex; align-items: center; gap: 8px;
            font-size: 12px; color: #27ae60; margin-bottom: 12px;
            justify-content: center;
        }
        .chef-status .dot {
            width: 8px; height: 8px; background: #27ae60;
            border-radius: 50%; animation: pulse 1.5s infinite;
        }
        @keyframes pulse { 0%, 100% { opacity: 1; } 50% { opacity: 0.4; } }
        .new-order-btn {
            margin-top: 20px; padding: 12px 32px;
            border: 2px solid #2c3e50; border-radius: 10px;
            background: transparent; color: #2c3e50;
            font-weight: 700; cursor: pointer; font-size: 14px;
        }
        .new-order-btn:hover { background: #2c3e50; color: white; }
        .customer-form {
            background: white; border: 1px solid #e0e0e0;
            border-radius: 16px; padding: 20px; margin-bottom: 20px;
            display: none;
        }
        .customer-form.visible { display: block; }
        .form-group { margin-bottom: 14px; }
        .form-group label {
            display: block; font-weight: 600; font-size: 14px;
            margin-bottom: 6px; color: #333;
        }
        .form-group input, .form-group textarea {
            width: 100%; padding: 12px; border: 1px solid #ddd;
            border-radius: 10px; font-size: 14px; font-family: inherit;
        }
        .form-group input:focus, .form-group textarea:focus {
            outline: none; border-color: #c0392b;
        }
        .submit-order-btn {
            width: 100%; padding: 14px; border: none; border-radius: 12px;
            background: #27ae60; color: white; font-weight: 800;
            font-size: 16px; cursor: pointer; margin-top: 10px;
        }
        .submit-order-btn:hover { background: #2ecc71; }
        .back-btn {
            background: #888; color: white; border: none;
            padding: 10px 20px; border-radius: 8px;
            cursor: pointer; font-weight: 600; margin-bottom: 10px;
        }
        .back-btn:hover { background: #666; }
        .top-bar {
            display: flex; justify-content: space-between;
            align-items: center; padding: 10px 16px;
            background: white; border-bottom: 1px solid #eee;
        }
        .top-bar a {
            color: #c0392b; text-decoration: none;
            font-weight: 600; font-size: 14px;
        }
        .top-bar a:hover { text-decoration: underline; }
    </style>
</head>
<body>
    <div class="top-bar">
        <div></div>
        <a href="/chef">🧑‍🍳 دخول الشيف</a>
    </div>
    
    <div class="contact-bar">
        📞 للتواصل والطلب: <a href="tel:{{ contact_phone }}">{{ contact_phone }}</a>
        | <a href="https://wa.me/20{{ contact_phone.lstrip('0') }}" target="_blank">واتساب 💬</a>
    </div>
    
    <div class="header">
        <div style="font-size:48px; margin-bottom:8px;">🍰</div>
        <h1>حلوياتك عندنا</h1>
        <p>أشهى الحلويات الشرقية والغربية — توصيل سريع للباب</p>
    </div>
    
    <div class="container">
        <div class="tabs" id="tabs"></div>
        <div class="menu-grid" id="menuGrid"></div>
        
        <div class="cart-panel" id="cartPanel">
            <div class="cart-title">🛒 طلبك</div>
            <div id="cartItems">
                <div class="empty-cart">سلة الطلبات فارغة — اختر حلوياتك المفضلة!</div>
            </div>
            <div id="cartFooter" style="display:none;">
                <div class="cart-total">
                    <span>الإجمالي:</span>
                    <span id="totalPrice">0 جنيه</span>
                </div>
                <div class="limit-warning" id="limitWarning">
                    ⚠️ تجاوزت الحد الأقصى (150 جنيه). أزل بعض الأصناف للمتابعة.
                </div>
                <button class="order-btn" id="orderBtn" onclick="showCustomerForm()">
                    📤 متابعة الطلب
                </button>
            </div>
        </div>
        
        <div class="customer-form" id="customerForm">
            <button class="back-btn" onclick="hideCustomerForm()">← رجوع للسلة</button>
            <h3 style="margin-bottom:16px;">📝 بيانات التوصيل</h3>
            <div class="form-group">
                <label>الاسم</label>
                <input type="text" id="customerName" placeholder="اكتب اسمك">
            </div>
            <div class="form-group">
                <label>رقم التليفون</label>
                <input type="tel" id="customerPhone" placeholder="مثال: 01012345678">
            </div>
            <div class="form-group">
                <label>العنوان</label>
                <textarea id="customerAddress" rows="3" placeholder="اكتب عنوان التوصيل بالتفصيل"></textarea>
            </div>
            <button class="submit-order-btn" onclick="submitOrder()">
                ✅ تأكيد وإرسال الطلب للشيف
            </button>
        </div>
        
        <div class="order-confirmation" id="orderConfirmation">
            <div class="big-emoji">✅</div>
            <h2>تم إرسال طلبك بنجاح!</h2>
            <p>وصل للشيف على طول 🧑‍🍳</p>
            <div class="chef-status">
                <div class="dot"></div>
                <span>الشيف بيحضر طلبك دلوقتي</span>
            </div>
            <p style="margin-top:12px; font-size:13px;">
                إجمالي الطلب: <strong id="finalTotal">0 جنيه</strong>
            </p>
            <p style="margin-top:8px; font-size:13px; color:#666;">
                رقم الطلب: <strong id="orderId">-</strong>
            </p>
            <p style="margin-top:8px; font-size:13px;">
                📞 للاستفسار: <a href="tel:{{ contact_phone }}" style="color:#c0392b; font-weight:700;">{{ contact_phone }}</a>
            </p>
            <button class="new-order-btn" onclick="resetOrder()">🔄 طلب جديد</button>
        </div>
    </div>
    
    <div class="notification" id="notification"></div>
    
    <script>
        const MAX_BUDGET = 150;
        const categories = {{ categories | tojson }};
        const menuItems = {{ menu_items | tojson }};
        let cart = {};
        let activeCategory = 'all';
        
        function renderTabs() {
            const tabsEl = document.getElementById('tabs');
            tabsEl.innerHTML = categories.map(cat => `
                <button class="tab ${cat.id === activeCategory ? 'active' : ''}" 
                        onclick="setCategory('${cat.id}')">${cat.name}</button>
            `).join('');
        }
        
        function setCategory(catId) {
            activeCategory = catId;
            renderTabs();
            renderMenu();
        }
        
        function renderMenu() {
            const grid = document.getElementById('menuGrid');
            const items = activeCategory === 'all' 
                ? menuItems 
                : menuItems.filter(i => i.category === activeCategory);
            
            grid.innerHTML = items.map(item => {
                const inCart = cart[item.id] || 0;
                const currentTotal = getCartTotal();
                const canAdd = currentTotal + item.price <= MAX_BUDGET;
                
                return `
                    <div class="item-card">
                        <span class="item-emoji">${item.emoji}</span>
                        <div class="item-name">${item.name}</div>
                        <div class="item-desc">${item.desc}</div>
                        <div class="item-price">${item.price} جنيه</div>
                        <button class="add-btn" 
                                onclick="addToCart(${item.id})"
                                ${!canAdd ? 'disabled' : ''}>
                            ${inCart > 0 ? '✓ ' + inCart + ' في السلة' : '+ أضف للسلة'}
                        </button>
                    </div>
                `;
            }).join('');
        }
        
        function getCartTotal() {
            return Object.entries(cart).reduce((sum, [id, qty]) => {
                const item = menuItems.find(i => i.id == id);
                return sum + (item ? item.price * qty : 0);
            }, 0);
        }
        
        function addToCart(itemId) {
            const item = menuItems.find(i => i.id === itemId);
            const currentTotal = getCartTotal();
            
            if (currentTotal + item.price > MAX_BUDGET) {
                showNotification('⚠️ تجاوزت الحد الأقصى (150 جنيه)');
                return;
            }
            
            cart[itemId] = (cart[itemId] || 0) + 1;
            showNotification('✅ تم إضافة ' + item.name);
            renderMenu();
            renderCart();
        }
        
        function removeFromCart(itemId) {
            if (cart[itemId] > 1) {
                cart[itemId]--;
            } else {
                delete cart[itemId];
            }
            renderMenu();
            renderCart();
        }
        
        function renderCart() {
            const cartItemsEl = document.getElementById('cartItems');
            const cartFooter = document.getElementById('cartFooter');
            const totalEl = document.getElementById('totalPrice');
            const limitWarning = document.getElementById('limitWarning');
            const orderBtn = document.getElementById('orderBtn');
            
            const entries = Object.entries(cart);
            const total = getCartTotal();
            
            if (entries.length === 0) {
                cartItemsEl.innerHTML = '<div class="empty-cart">سلة الطلبات فارغة — اختر حلوياتك المفضلة!</div>';
                cartFooter.style.display = 'none';
                return;
            }
            
            cartItemsEl.innerHTML = entries.map(([id, qty]) => {
                const item = menuItems.find(i => i.id == id);
                return `
                    <div class="cart-item">
                        <div class="cart-item-info">
                            <div class="cart-item-name">${item.emoji} ${item.name}</div>
                            <div class="cart-item-price">${item.price} جنيه × ${qty} = ${item.price * qty} جنيه</div>
                        </div>
                        <div class="qty-controls">
                            <button class="qty-btn" onclick="removeFromCart(${id})">−</button>
                            <span style="font-weight:700; min-width:20px; text-align:center;">${qty}</span>
                            <button class="qty-btn" onclick="addToCart(${id})">+</button>
                        </div>
                    </div>
                `;
            }).join('');
            
            totalEl.textContent = total + ' جنيه';
            cartFooter.style.display = 'block';
            
            if (total >= MAX_BUDGET) {
                limitWarning.classList.add('visible');
                orderBtn.disabled = total > MAX_BUDGET;
            } else {
                limitWarning.classList.remove('visible');
                orderBtn.disabled = false;
            }
        }
        
        function showCustomerForm() {
            document.getElementById('cartPanel').style.display = 'none';
            document.getElementById('menuGrid').style.display = 'none';
            document.getElementById('tabs').style.display = 'none';
            document.getElementById('customerForm').classList.add('visible');
        }
        
        function hideCustomerForm() {
            document.getElementById('customerForm').classList.remove('visible');
            document.getElementById('cartPanel').style.display = 'block';
            document.getElementById('menuGrid').style.display = 'grid';
            document.getElementById('tabs').style.display = 'flex';
        }
        
        async function submitOrder() {
            const name = document.getElementById('customerName').value.trim();
            const phone = document.getElementById('customerPhone').value.trim();
            const address = document.getElementById('customerAddress').value.trim();
            
            if (!name || !phone || !address) {
                showNotification('⚠️ املأ جميع البيانات');
                return;
            }
            
            const total = getCartTotal();
            
            try {
                const response = await fetch('/api/order', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({
                        items: cart,
                        customer_name: name,
                        phone: phone,
                        address: address,
                        total: total
                    })
                });
                
                const data = await response.json();
                
                if (data.success) {
                    document.getElementById('customerForm').classList.remove('visible');
                    const confirmation = document.getElementById('orderConfirmation');
                    confirmation.classList.add('visible');
                    
                    document.getElementById('finalTotal').textContent = total + ' جنيه';
                    document.getElementById('orderId').textContent = data.order_id;
                    
                    showNotification('🧑‍🍳 وصل للشيف! بيحضر طلبك دلوقتي');
                } else {
                    showNotification('❌ ' + data.message);
                }
            } catch (e) {
                showNotification('❌ حدث خطأ، حاول مرة أخرى');
            }
        }
        
        function resetOrder() {
            cart = {};
            activeCategory = 'all';
            
            document.getElementById('menuGrid').style.display = 'grid';
            document.getElementById('tabs').style.display = 'flex';
            document.getElementById('cartPanel').style.display = 'block';
            document.getElementById('orderConfirmation').classList.remove('visible');
            document.getElementById('customerForm').classList.remove('visible');
            
            document.getElementById('customerName').value = '';
            document.getElementById('customerPhone').value = '';
            document.getElementById('customerAddress').value = '';
            
            renderTabs();
            renderMenu();
            renderCart();
        }
        
        function showNotification(msg) {
            const notif = document.getElementById('notification');
            notif.textContent = msg;
            notif.classList.add('show');
            setTimeout(() => notif.classList.remove('show'), 2500);
        }
        
        renderTabs();
        renderMenu();
        renderCart();
    </script>
</body>
</html>
"""

# ==================== AUTH PAGES ====================
LOGIN_HTML = """
<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>تسجيل الدخول - حلوياتك عندنا</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            font-family: 'Segoe UI', Tahoma, sans-serif;
            background: #f5f5f5;
            direction: rtl;
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
        }
        .login-box {
            background: white;
            padding: 40px;
            border-radius: 20px;
            box-shadow: 0 10px 40px rgba(0,0,0,0.1);
            width: 100%;
            max-width: 400px;
        }
        .login-box h2 {
            text-align: center;
            color: #2c3e50;
            margin-bottom: 8px;
            font-size: 24px;
        }
        .login-box .subtitle {
            text-align: center;
            color: #888;
            margin-bottom: 30px;
            font-size: 14px;
        }
        .form-group { margin-bottom: 20px; }
        .form-group label {
            display: block;
            font-weight: 600;
            font-size: 14px;
            margin-bottom: 8px;
            color: #333;
        }
        .form-group input {
            width: 100%;
            padding: 14px;
            border: 1px solid #ddd;
            border-radius: 12px;
            font-size: 15px;
            font-family: inherit;
        }
        .form-group input:focus {
            outline: none;
            border-color: #c0392b;
        }
        .login-btn {
            width: 100%;
            padding: 14px;
            border: none;
            border-radius: 12px;
            background: #c0392b;
            color: white;
            font-weight: 800;
            font-size: 16px;
            cursor: pointer;
            margin-top: 10px;
        }
        .login-btn:hover { background: #e74c3c; }
        .back-link {
            display: block;
            text-align: center;
            margin-top: 20px;
            color: #888;
            text-decoration: none;
            font-size: 14px;
        }
        .back-link:hover { color: #c0392b; }
        .alert {
            padding: 12px;
            border-radius: 10px;
            margin-bottom: 20px;
            font-size: 14px;
        }
        .alert.error {
            background: #ffebee;
            color: #c0392b;
            border: 1px solid #ef9a9a;
        }
        .alert.success {
            background: #e8f5e9;
            color: #27ae60;
            border: 1px solid #a5d6a7;
        }
    </style>
</head>
<body>
    <div class="login-box">
        <h2>🧁 حلوياتك عندنا</h2>
        <p class="subtitle">تسجيل الدخول للموظفين</p>
        
        {% with messages = get_flashed_messages(with_categories=true) %}
            {% if messages %}
                {% for category, message in messages %}
                    <div class="alert {{ category }}">{{ message }}</div>
                {% endfor %}
            {% endif %}
        {% endwith %}
        
        <form method="POST" action="/login">
            <div class="form-group">
                <label>اسم المستخدم</label>
                <input type="text" name="username" placeholder="اسم المستخدم" required>
            </div>
            <div class="form-group">
                <label>كلمة المرور</label>
                <input type="password" name="password" placeholder="كلمة المرور" required>
            </div>
            <button type="submit" class="login-btn">🔐 دخول</button>
        </form>
        <a href="/" class="back-link">← الرجوع للموقع</a>
    </div>
</body>
</html>
"""

# ==================== CHEF DASHBOARD ====================
CHEF_HTML = """
<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>لوحة الشيف - حلوياتك عندنا</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            font-family: 'Segoe UI', Tahoma, sans-serif;
            background: #f5f5f5;
            direction: rtl;
        }
        .top-bar {
            background: #2c3e50;
            color: white;
            padding: 16px 24px;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
        .top-bar h1 { font-size: 20px; color: #f1c40f; }
        .top-bar .user-info {
            display: flex;
            align-items: center;
            gap: 16px;
            font-size: 14px;
        }
        .top-bar a {
            color: white;
            text-decoration: none;
            background: #c0392b;
            padding: 8px 16px;
            border-radius: 8px;
            font-size: 13px;
        }
        .top-bar a:hover { background: #e74c3c; }
        .container {
            max-width: 1100px;
            margin: 0 auto;
            padding: 24px;
        }
        .stats {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
            gap: 16px;
            margin-bottom: 24px;
        }
        .stat-card {
            background: white;
            border-radius: 14px;
            padding: 20px;
            border: 1px solid #e0e0e0;
        }
        .stat-card .label {
            font-size: 13px;
            color: #888;
            margin-bottom: 8px;
        }
        .stat-card .value {
            font-size: 28px;
            font-weight: 800;
            color: #2c3e50;
        }
        .stat-card.pending .value { color: #f39c12; }
        .stat-card.completed .value { color: #27ae60; }
        .stat-card.revenue .value { color: #c0392b; }
        .section-title {
            font-size: 18px;
            font-weight: 700;
            margin-bottom: 16px;
            color: #2c3e50;
        }
        .orders-table {
            background: white;
            border-radius: 14px;
            border: 1px solid #e0e0e0;
            overflow: hidden;
        }
        .order-row {
            display: grid;
            grid-template-columns: 80px 1fr 120px 100px 140px;
            gap: 12px;
            padding: 16px;
            border-bottom: 1px solid #eee;
            align-items: center;
        }
        .order-row.header {
            background: #f8f9fa;
            font-weight: 700;
            font-size: 13px;
            color: #666;
        }
        .order-row:last-child { border-bottom: none; }
        .order-id { font-weight: 800; color: #c0392b; }
        .order-items { font-size: 13px; color: #555; line-height: 1.6; }
        .order-customer { font-size: 13px; color: #666; }
        .order-total { font-weight: 700; color: #c0392b; font-size: 15px; }
        .status-badge {
            padding: 6px 12px;
            border-radius: 20px;
            font-size: 12px;
            font-weight: 700;
            text-align: center;
        }
        .status-pending {
            background: #fff3e0;
            color: #e65100;
        }
        .status-completed {
            background: #e8f5e9;
            color: #2e7d32;
        }
        .action-btn {
            padding: 8px 14px;
            border: none;
            border-radius: 8px;
            cursor: pointer;
            font-weight: 600;
            font-size: 13px;
        }
        .complete-btn {
            background: #27ae60;
            color: white;
        }
        .complete-btn:hover { background: #2ecc71; }
        .empty-state {
            text-align: center;
            padding: 60px 20px;
            color: #888;
        }
        .empty-state .emoji { font-size: 48px; margin-bottom: 12px; }
        .refresh-btn {
            background: #2c3e50;
            color: white;
            border: none;
            padding: 10px 20px;
            border-radius: 8px;
            cursor: pointer;
            font-weight: 600;
            margin-bottom: 16px;
        }
        .refresh-btn:hover { background: #34495e; }
        .alert {
            padding: 12px 16px;
            border-radius: 10px;
            margin-bottom: 16px;
            font-size: 14px;
        }
        .alert.success {
            background: #e8f5e9;
            color: #27ae60;
            border: 1px solid #a5d6a7;
        }
        .phone-link {
            color: #c0392b;
            text-decoration: none;
            font-weight: 600;
        }
        .phone-link:hover { text-decoration: underline; }
        @media (max-width: 768px) {
            .order-row {
                grid-template-columns: 1fr;
                gap: 8px;
            }
            .order-row.header { display: none; }
        }
    </style>
</head>
<body>
    <div class="top-bar">
        <h1>🧑‍🍳 لوحة تحكم الشيف</h1>
        <div class="user-info">
            <span>مرحباً، {{ user_name }}</span>
            <a href="/logout">🚪 خروج</a>
        </div>
    </div>
    
    <div class="container">
        {% with messages = get_flashed_messages(with_categories=true) %}
            {% if messages %}
                {% for category, message in messages %}
                    <div class="alert {{ category }}">{{ message }}</div>
                {% endfor %}
            {% endif %}
        {% endwith %}
        
        <div class="stats">
            <div class="stat-card pending">
                <div class="label">طلبات قيد التحضير</div>
                <div class="value">{{ pending_count }}</div>
            </div>
            <div class="stat-card completed">
                <div class="label">طلبات مكتملة</div>
                <div class="value">{{ completed_count }}</div>
            </div>
            <div class="stat-card revenue">
                <div class="label">إجمالي المبيعات</div>
                <div class="value">{{ total_revenue }} ج</div>
            </div>
            <div class="stat-card">
                <div class="label">إجمالي الطلبات</div>
                <div class="value">{{ total_orders }}</div>
            </div>
        </div>
        
        <div style="display:flex; justify-content:space-between; align-items:center; margin-bottom:16px;">
            <h3 class="section-title">📋 الطلبات الجديدة</h3>
            <button class="refresh-btn" onclick="location.reload()">🔄 تحديث</button>
        </div>
        
        {% if pending_orders %}
        <div class="orders-table">
            <div class="order-row header">
                <div>رقم الطلب</div>
                <div>الأصناف والعميل</div>
                <div>العنوان</div>
                <div>الإجمالي</div>
                <div>الإجراء</div>
            </div>
            {% for order in pending_orders %}
            <div class="order-row">
                <div class="order-id">#{{ order.id }}</div>
                <div>
                    <div class="order-items">{{ order.items }}</div>
                    <div class="order-customer" style="margin-top:4px;">
                        👤 {{ order.customer_name }} | 
                        <a href="tel:{{ order.phone }}" class="phone-link">📞 {{ order.phone }}</a>
                    </div>
                </div>
                <div class="order-customer">📍 {{ order.address }}</div>
                <div class="order-total">{{ order.total }} ج</div>
                <div>
                    <form method="POST" action="/chef/complete/{{ order.id }}" style="display:inline;">
                        <button type="submit" class="action-btn complete-btn">✅ تم التحضير</button>
                    </form>
                </div>
            </div>
            {% endfor %}
        </div>
        {% else %}
        <div class="orders-table">
            <div class="empty-state">
                <div class="emoji">☕</div>
                <p>مفيش طلبات جديدة دلوقتي. استراحة الشيف! 😊</p>
            </div>
        </div>
        {% endif %}
        
        {% if completed_orders %}
        <h3 class="section-title" style="margin-top:30px;">📦 الطلبات المكتملة (آخر 20)</h3>
        <div class="orders-table">
            <div class="order-row header">
                <div>رقم الطلب</div>
                <div>الأصناف والعميل</div>
                <div>الوقت</div>
                <div>الإجمالي</div>
                <div>الحالة</div>
            </div>
            {% for order in completed_orders %}
            <div class="order-row">
                <div class="order-id">#{{ order.id }}</div>
                <div>
                    <div class="order-items">{{ order.items }}</div>
                    <div class="order-customer" style="margin-top:4px;">👤 {{ order.customer_name }}</div>
                </div>
                <div class="order-customer">{{ order.completed_at }}</div>
                <div class="order-total">{{ order.total }} ج</div>
                <div><span class="status-badge status-completed">✅ مكتمل</span></div>
            </div>
            {% endfor %}
        </div>
        {% endif %}
    </div>
</body>
</html>
"""

# ==================== ROUTES ====================

@app.route('/')
def index():
    return render_template_string(CUSTOMER_HTML, 
                                  categories=categories, 
                                  menu_items=menu_items,
                                  contact_phone=CONTACT_PHONE)

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        username = request.form.get('username')
        password = request.form.get('password')
        
        conn = sqlite3.connect(DB_PATH)
        c = conn.cursor()
        c.execute("SELECT id, name, role FROM users WHERE username = ? AND password = ?", 
                  (username, password))
        user = c.fetchone()
        conn.close()
        
        if user:
            session['user_id'] = user[0]
            session['user_name'] = user[1]
            session['user_role'] = user[2]
            flash('تم تسجيل الدخول بنجاح!', 'success')
            if user[2] == 'chef':
                return redirect(url_for('chef_dashboard'))
            return redirect(url_for('index'))
        else:
            flash('اسم المستخدم أو كلمة المرور غير صحيحة', 'error')
    
    return render_template_string(LOGIN_HTML)

@app.route('/logout')
def logout():
    session.clear()
    flash('تم تسجيل الخروج', 'success')
    return redirect(url_for('index'))

@app.route('/chef')
@chef_required
def chef_dashboard():
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    
    # Pending orders
    c.execute("""SELECT id, customer_name, phone, address, items, total, created_at 
                 FROM orders WHERE status = 'pending' ORDER BY created_at DESC""")
    pending = c.fetchall()
    
    # Completed orders (last 20)
    c.execute("""SELECT id, customer_name, items, total, completed_at 
                 FROM orders WHERE status = 'completed' 
                 ORDER BY completed_at DESC LIMIT 20""")
    completed = c.fetchall()
    
    # Stats
    c.execute("SELECT COUNT(*) FROM orders WHERE status = 'pending'")
    pending_count = c.fetchone()[0]
    
    c.execute("SELECT COUNT(*) FROM orders WHERE status = 'completed'")
    completed_count = c.fetchone()[0]
    
    c.execute("SELECT COALESCE(SUM(total), 0) FROM orders WHERE status = 'completed'")
    total_revenue = c.fetchone()[0]
    
    c.execute("SELECT COUNT(*) FROM orders")
    total_orders = c.fetchone()[0]
    
    conn.close()
    
    pending_orders = []
    for row in pending:
        pending_orders.append({
            'id': row[0],
            'customer_name': row[1],
            'phone': row[2],
            'address': row[3],
            'items': row[4],
            'total': row[5],
            'created_at': row[6]
        })
    
    completed_orders = []
    for row in completed:
        completed_orders.append({
            'id': row[0],
            'customer_name': row[1],
            'items': row[2],
            'total': row[3],
            'completed_at': row[4]
        })
    
    return render_template_string(CHEF_HTML,
                                  user_name=session.get('user_name', 'الشيف'),
                                  pending_orders=pending_orders,
                                  completed_orders=completed_orders,
                                  pending_count=pending_count,
                                  completed_count=completed_count,
                                  total_revenue=total_revenue,
                                  total_orders=total_orders)

@app.route('/chef/complete/<int:order_id>', methods=['POST'])
@chef_required
def complete_order(order_id):
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute("""UPDATE orders SET status = 'completed', completed_at = ? WHERE id = ?""",
              (datetime.now().strftime('%Y-%m-%d %H:%M'), order_id))
    conn.commit()
    conn.close()
    flash('تم إكمال الطلب بنجاح!', 'success')
    return redirect(url_for('chef_dashboard'))

@app.route('/api/order', methods=['POST'])
def place_order():
    data = request.get_json()
    items = data.get('items', {})
    customer_name = data.get('customer_name', '')
    phone = data.get('phone', '')
    address = data.get('address', '')
    total = data.get('total', 0)
    
    if total > MAX_BUDGET:
        return jsonify({'success': False, 'message': 'تجاوزت الحد الأقصى 150 جنيه'}), 400
    
    # Format items string
    items_list = []
    for item_id, qty in items.items():
        item = next((i for i in menu_items if i['id'] == int(item_id)), None)
        if item:
            items_list.append(f"{item['name']} x{qty}")
    items_str = ' | '.join(items_list)
    
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute("""INSERT INTO orders (customer_name, phone, address, items, total, status)
                 VALUES (?, ?, ?, ?, ?, 'pending')""",
              (customer_name, phone, address, items_str, total))
    order_id = c.lastrowid
    conn.commit()
    conn.close()
    
    return jsonify({
        'success': True,
        'message': 'تم إرسال الطلب للشيف بنجاح!',
        'total': total,
        'order_id': f'ORD-{order_id}'
    })


if __name__ == '__main__':
    app.run(debug=True, host='0.0.0.0', port=5000)
'''

# Save the file
output_path = '/mnt/agents/output/halawiyatna_app.py'
with open(output_path, 'w', encoding='utf-8') as f:
    f.write(app_code)

print(f"✅ File saved successfully to: {output_path}")
print(f"📄 File size: {len(app_code)} characters")
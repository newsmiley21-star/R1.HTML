<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>X-PRESS RESTO | Terminal</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <link href="https://fonts.googleapis.com/css2?family=Outfit:wght@300;400;600;700;800&display=swap" rel="stylesheet">
    <script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-firestore-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-auth-compat.js"></script>
    
    <style>
        body { font-family: 'Outfit', sans-serif; background-color: #f1f5f9; }
        .glass-card { background: rgba(255, 255, 255, 0.9); backdrop-filter: blur(20px); border: 1px solid rgba(255, 255, 255, 0.3); }
        .mode-active { background: #1e293b; color: white; }
        .no-scrollbar::-webkit-scrollbar { display: none; }
        @media print {
            body * { visibility: hidden; }
            #printable-ticket, #printable-ticket * { visibility: visible; }
            #printable-ticket { position: absolute; left: 0; top: 0; width: 72mm; color: black; }
        }
    </style>
</head>
<body class="pb-32">

    <!-- Écran de Connexion -->
    <div id="login-screen" class="fixed inset-0 z-[200] flex items-center justify-center p-4 bg-slate-100">
        <div class="glass-card p-8 rounded-[2.5rem] w-full max-w-sm shadow-2xl">
            <h1 class="text-3xl font-black mb-1">X-PRESS <span class="text-green-600">RESTO</span></h1>
            <p class="text-slate-500 font-bold mb-8 text-sm">Connexion terminal</p>
            <input type="email" id="email" placeholder="Email" class="w-full bg-slate-100 p-4 rounded-2xl font-bold mb-4">
            <input type="password" id="password" placeholder="Mot de passe" class="w-full bg-slate-100 p-4 rounded-2xl font-bold mb-6">
            <button onclick="login()" class="w-full bg-slate-900 text-white py-4 rounded-2xl font-black text-sm uppercase shadow-xl">Se connecter</button>
        </div>
    </div>

    <!-- Application -->
    <div id="app-content" style="display:none;">
        <header class="fixed top-0 inset-x-0 z-50 p-4">
            <div class="glass-card mx-auto max-w-lg rounded-[2.5rem] p-4 flex items-center justify-between shadow-2xl">
                <div>
                    <h1 class="text-xl font-black tracking-tighter">X-PRESS <span class="text-green-600">RESTO</span></h1>
                    <p id="total-display" class="text-[10px] font-bold text-slate-500 uppercase tracking-widest">Total: 0 FCFA</p>
                </div>
                <div class="flex gap-3">
                    <button onclick="resetCart()" class="text-[10px] font-bold text-slate-400">ANNULER</button>
                    <button id="export-btn" onclick="exportData()" class="hidden bg-green-600 text-white px-2 py-1 rounded-lg text-[9px] font-bold">EXPORT</button>
                    <button onclick="logout()" class="text-[10px] font-bold text-red-500 underline">EXIT</button>
                </div>
            </div>
        </header>

        <main class="pt-28 px-4 max-w-lg mx-auto">
            <div class="flex p-1 bg-white rounded-3xl shadow-sm mb-4 border border-slate-100">
                <button onclick="setOrderMode('Sur Place')" id="mode-sur-place" class="mode-active flex-1 py-3 rounded-2xl font-bold text-sm">Sur Place</button>
                <button onclick="setOrderMode('À Emporter')" id="mode-emporter" class="flex-1 py-3 rounded-2xl font-bold text-sm">À Emporter</button>
            </div>
            <div id="table-input-container" class="mb-6">
                <input type="number" id="table-number" placeholder="N° de Table" class="w-full bg-white p-4 rounded-2xl font-bold text-sm shadow-sm border border-slate-100">
            </div>
            <div id="menu-grid" class="grid grid-cols-2 gap-4"></div>
        </main>

        <div class="fixed bottom-0 inset-x-0 p-4 z-40">
            <button onclick="openPaymentModal()" class="w-full max-w-lg mx-auto block bg-slate-900 text-white py-5 rounded-[2rem] font-black text-sm uppercase shadow-xl">Valider & Payer</button>
        </div>
    </div>

    <!-- Modal Paiement -->
    <div id="payment-modal" class="fixed inset-0 z-[100] hidden items-center justify-center p-4 bg-slate-900/50 backdrop-blur-sm">
        <div class="bg-white rounded-[2.5rem] p-8 w-full max-w-sm shadow-2xl">
            <h2 class="text-2xl font-black mb-4">Paiement</h2>
            <input type="number" id="cash-input" oninput="checkPayment()" class="w-full bg-slate-100 p-4 rounded-2xl font-black text-xl mb-4" placeholder="Montant remis (FCFA)">
            <div class="flex gap-3">
                <button onclick="closePayment()" class="flex-1 py-4 rounded-2xl font-bold bg-slate-100 text-slate-600 text-sm">RETOUR</button>
                <button id="confirm-pay-btn" onclick="finalizeOrder()" disabled class="flex-1 py-4 rounded-2xl font-black text-sm bg-slate-200 text-slate-400">CONFIRMER</button>
            </div>
        </div>
    </div>
    <div id="printable-ticket" class="hidden"></div>

    <script>
        const firebaseConfig = { apiKey: "AIzaSyCEJJGhcyYWqmeI9D_lwk_qgE2J2GZhIlg", authDomain: "communautedugabon.firebaseapp.com", projectId: "communautedugabon", storageBucket: "communautedugabon.firebasestorage.app", messagingSenderId: "647862371022", appId: "1:647862371022:web:b209bfc8eb81accb1fc69f" };
        firebase.initializeApp(firebaseConfig);
        const db = firebase.firestore();
        const auth = firebase.auth();

        auth.onAuthStateChanged(user => {
            if (user) { 
                document.getElementById('login-screen').style.display = 'none'; 
                document.getElementById('app-content').style.display = 'block'; 
                if(user.email === 'admin@x-press.com') document.getElementById('export-btn').classList.remove('hidden');
                renderMenu(); 
            } else { location.reload(); }
        });

        function login() { auth.signInWithEmailAndPassword(document.getElementById('email').value, document.getElementById('password').value).catch(e => alert(e.message)); }
        function logout() { auth.signOut(); }

        const products = [{ id: 1, name: "Burger Premium", price: 3500 }, { id: 2, name: "Pizza", price: 4500 }];
        let cart = {}; let orderMode = "Sur Place";

        function renderMenu() {
            const grid = document.getElementById('menu-grid');
            grid.innerHTML = products.map(p => `
                <div class="glass-card rounded-[2rem] p-4 text-center">
                    <h3 class="font-bold text-xs truncate">${p.name}</h3>
                    <p class="text-green-600 font-black text-[10px] mb-2">${p.price.toLocaleString()} FCFA</p>
                    <div class="flex items-center justify-between bg-slate-100 p-1 rounded-xl">
                        <button onclick="updateQty(${p.id}, -1)" class="w-8 h-8 rounded-lg bg-white">-</button>
                        <span id="qty-${p.id}" class="font-black text-xs">${cart[p.id] || 0}</span>
                        <button onclick="updateQty(${p.id}, 1)" class="w-8 h-8 rounded-lg bg-green-500 text-white">+</button>
                    </div>
                </div>
            `).join('');
        }

        function updateQty(id, delta) { cart[id] = Math.max(0, (cart[id] || 0) + delta); document.getElementById(`qty-${id}`).innerText = cart[id]; updateTotal(); }
        
        function resetCart() { 
            cart = {}; 
            document.querySelectorAll('[id^="qty-"]').forEach(el => el.innerText = '0'); 
            updateTotal(); 
        }

        function updateTotal() {
            let total = 0;
            for (const [id, qty] of Object.entries(cart)) {
                const p = products.find(prod => prod.id == id);
                if (p) total += (p.price * qty);
            }
            document.getElementById('total-display').innerText = `Total: ${total.toLocaleString()} FCFA`;
            return total;
        }

        function openPaymentModal() { if (updateTotal() > 0) document.getElementById('payment-modal').classList.replace('hidden', 'flex'); }
        function closePayment() { document.getElementById('payment-modal').classList.replace('flex', 'hidden'); }
        
        function checkPayment() { if(parseInt(document.getElementById('cash-input').value) >= updateTotal()) document.getElementById('confirm-pay-btn').disabled = false; }
        
        async function finalizeOrder() {
            const total = updateTotal();
            const orderData = { items: cart, total: total, date: new Date().toLocaleString(), mode: orderMode, table: document.getElementById('table-number').value };
            await db.collection('orders').add(orderData);
            window.print();
            location.reload();
        }

        async function exportData() {
            const snapshot = await db.collection('orders').get();
            let csv = "\uFEFFDate,Mode,Table,Total,Produit,Quantité,Prix\n";
            snapshot.forEach(doc => { const d = doc.data(); for (const [id, qty] of Object.entries(d.items)) { if(qty > 0) { const p = products.find(prod => prod.id == id); csv += `${d.date},${d.mode},${d.table},${d.total},${p.name},${qty},${p.price}\n`; } } });
            const blob = new Blob([csv], { type: 'text/csv;charset=utf-8;' });
            const url = window.URL.createObjectURL(blob);
            const a = document.createElement('a'); a.href = url; a.download = 'rapport.csv'; a.click();
        }

        function setOrderMode(mode) {
            orderMode = mode;
            document.getElementById('mode-sur-place').classList.toggle('mode-active', mode === 'Sur Place');
            document.getElementById('mode-emporter').classList.toggle('mode-active', mode === 'À Emporter');
            document.getElementById('table-input-container').style.display = mode === 'Sur Place' ? 'block' : 'none';
        }
    </script>
</body>
</html>

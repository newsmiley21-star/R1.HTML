<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>X-PRESS RESTO | Terminal de Vente</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <link href="https://fonts.googleapis.com/css2?family=Outfit:wght@300;400;600;700;800&display=swap" rel="stylesheet">
    
    <script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-firestore-compat.js"></script>
    
    <style>
        body { font-family: 'Outfit', sans-serif; background-color: #f1f5f9; }
        .glass-card { background: rgba(255, 255, 255, 0.9); backdrop-filter: blur(20px); border: 1px solid rgba(255, 255, 255, 0.3); }
        .mode-active { background: #1e293b; color: white; }
        .no-scrollbar::-webkit-scrollbar { display: none; }
        @media print { .no-print { display: none; } #printable-ticket { display: block; position: absolute; left: 0; top: 0; width: 80mm; } }
    </style>
</head>
<body class="pb-32">

    <script>
        // CONFIGURATION FIREBASE - REMPLISSEZ CES CHAMPS AVEC VOS CLÉS
        const firebaseConfig = {
            apiKey: "AIzaSyCEJJGhcyYWqmeI9D_lwk_qgE2J2GZhIlg",
            authDomain: "communautedugabon.firebaseapp.com",
            projectId: "communautedugabon",
            storageBucket: "communautedugabon.firebasestorage.app",
            messagingSenderId: "647862371022",
            appId: "1:647862371022:web:b209bfc8eb81accb1fc69f"
        };
        
        firebase.initializeApp(firebaseConfig);
        const db = firebase.firestore();

        async function saveOrderToFirebase(orderData) {
            try {
                await db.collection("commandes").add({
                    ...orderData,
                    timestamp: firebase.firestore.FieldValue.serverTimestamp()
                });
            } catch (e) { console.error("Erreur Firebase : ", e); }
        }
    </script>

    <header class="fixed top-0 inset-x-0 z-50 p-4 no-print">
        <div class="glass-card mx-auto max-w-lg rounded-[2.5rem] p-4 flex items-center justify-between shadow-2xl">
            <div>
                <h1 class="text-xl font-black tracking-tighter text-slate-900">X-PRESS <span class="text-green-600">RESTO</span></h1>
                <p id="total-display" class="text-[10px] font-bold text-slate-500 uppercase tracking-widest">Total: 0 FCFA</p>
            </div>
            <button onclick="exportToExcel()" class="bg-slate-100 px-5 py-3 rounded-2xl font-black text-[10px] active:scale-95 transition-all">CSV</button>
        </div>
    </header>

    <main class="pt-28 px-4 max-w-lg mx-auto">
        <div class="flex p-1 bg-white rounded-3xl shadow-sm mb-4 border border-slate-100">
            <button onclick="setOrderMode('Sur Place')" id="mode-sur-place" class="mode-active flex-1 py-3 rounded-2xl font-bold text-sm transition-all">Sur Place</button>
            <button onclick="setOrderMode('À Emporter')" id="mode-emporter" class="flex-1 py-3 rounded-2xl font-bold text-sm transition-all">À Emporter</button>
        </div>

        <!-- Input Table caché par défaut -->
        <div id="table-input-container" class="mb-6">
            <input type="number" id="table-number" placeholder="Numéro de la table" class="w-full bg-white p-4 rounded-2xl font-bold text-sm shadow-sm border border-slate-100 focus:outline-none focus:ring-2 focus:ring-green-500">
        </div>

        <div id="category-nav" class="flex gap-3 mb-6 overflow-x-auto no-scrollbar pb-2"></div>
        <div id="menu-grid" class="grid grid-cols-2 gap-4"></div>
    </main>

    <div class="fixed bottom-0 inset-x-0 p-4 z-40 no-print">
        <div class="max-w-lg mx-auto">
            <button onclick="openPaymentModal()" class="w-full bg-slate-900 text-white py-5 rounded-[2rem] font-black text-sm uppercase shadow-xl hover:bg-slate-800 transition-all active:scale-95">
                Valider & Payer
            </button>
        </div>
    </div>

    <!-- Modal Paiement -->
    <div id="payment-modal" class="fixed inset-0 z-[100] hidden items-center justify-center p-4 bg-slate-900/50 backdrop-blur-sm">
        <div class="bg-white rounded-[2.5rem] p-8 w-full max-w-sm shadow-2xl">
            <h2 class="text-2xl font-black mb-1">Paiement</h2>
            <p id="modal-total" class="text-slate-500 font-bold mb-6"></p>
            <input type="number" id="cash-input" oninput="checkPayment()" class="w-full bg-slate-100 p-4 rounded-2xl font-black text-xl mb-4" placeholder="Montant reçu (FCFA)">
            <div id="payment-status" class="mb-6 font-bold text-sm"></div>
            <button id="confirm-pay-btn" onclick="finalizeOrder()" disabled class="w-full py-4 rounded-2xl font-black text-sm bg-slate-200 text-slate-400 transition-all">
                CONFIRMER L'IMPRESSION
            </button>
        </div>
    </div>

    <div id="printable-ticket" class="hidden text-black text-[10px] font-mono leading-tight"></div>

    <script>
        const products = [
            { id: 1, name: "Burger Premium", price: 3500, cat: "Burgers", img: "https://images.unsplash.com/photo-1568901346375-23c9450c58cd?w=200" },
            { id: 2, name: "Pizza Margherita", price: 4500, cat: "Pizzas", img: "https://images.unsplash.com/photo-1574071318508-1cdbab80d002?w=200" },
            { id: 3, name: "Salade César", price: 2500, cat: "Salades", img: "https://images.unsplash.com/photo-1550304943-4f24f54ddde9?w=200" },
            { id: 4, name: "Coca-Cola", price: 800, cat: "Boissons", img: "https://images.unsplash.com/photo-1629203851122-3726ecdf080e?w=200" }
        ];

        let cart = {};
        let currentCat = "Tous";
        let orderMode = "Sur Place";
        let orderCounter = 1001;

        function setOrderMode(mode) {
            orderMode = mode;
            document.getElementById('mode-sur-place').classList.toggle('mode-active', mode === 'Sur Place');
            document.getElementById('mode-emporter').classList.toggle('mode-active', mode === 'À Emporter');
            // Afficher le champ table uniquement pour "Sur Place"
            document.getElementById('table-input-container').style.display = mode === 'Sur Place' ? 'block' : 'none';
        }

        function renderCategories() {
            const categories = ["Tous", ...new Set(products.map(p => p.cat))];
            const nav = document.getElementById('category-nav');
            nav.innerHTML = categories.map(c => `
                <button onclick="setCategory('${c}')" class="px-5 py-2 rounded-2xl font-bold text-xs transition-all whitespace-nowrap ${currentCat === c ? 'bg-slate-900 text-white' : 'bg-white shadow-sm'}">
                    ${c}
                </button>
            `).join('');
        }

        function setCategory(cat) { currentCat = cat; renderCategories(); renderMenu(); }

        function renderMenu() {
            const grid = document.getElementById('menu-grid');
            const filtered = currentCat === "Tous" ? products : products.filter(p => p.cat === currentCat);
            grid.innerHTML = filtered.map(p => `
                <div class="glass-card rounded-[2rem] p-3 flex flex-col gap-2">
                    <img src="${p.img}" class="h-24 w-full object-cover rounded-2xl">
                    <div class="px-1">
                        <h3 class="font-bold text-xs truncate">${p.name}</h3>
                        <p class="text-green-600 font-black text-[10px]">${p.price.toLocaleString()} FCFA</p>
                    </div>
                    <div class="flex items-center justify-between bg-slate-100 p-1 rounded-2xl">
                        <button onclick="updateQty(${p.id}, -1)" class="w-8 h-8 rounded-xl bg-white shadow-sm font-bold text-xs">-</button>
                        <span id="qty-${p.id}" class="font-black text-xs">${cart[p.id] || 0}</span>
                        <button onclick="updateQty(${p.id}, 1)" class="w-8 h-8 rounded-xl bg-green-500 text-white shadow-sm font-bold text-xs">+</button>
                    </div>
                </div>
            `).join('');
        }

        function updateQty(id, delta) { cart[id] = Math.max(0, (cart[id] || 0) + delta); document.getElementById(`qty-${id}`).innerText = cart[id]; updateTotal(); }

        function updateTotal() {
            let total = 0;
            for (const [id, qty] of Object.entries(cart)) {
                const p = products.find(prod => prod.id == id);
                if (p) total += (p.price * qty);
            }
            document.getElementById('total-display').innerText = `Total: ${total.toLocaleString()} FCFA`;
            return total;
        }

        function openPaymentModal() {
            const total = updateTotal();
            if (total === 0) return;
            document.getElementById('modal-total').innerText = `Total à payer : ${total.toLocaleString()} FCFA`;
            document.getElementById('payment-modal').classList.replace('hidden', 'flex');
        }

        function checkPayment() {
            const total = updateTotal();
            const paid = parseInt(document.getElementById('cash-input').value) || 0;
            const btn = document.getElementById('confirm-pay-btn');
            const status = document.getElementById('payment-status');

            if (paid >= total) {
                status.innerHTML = `<span class="text-green-600">Rendu : ${(paid - total).toLocaleString()} FCFA</span>`;
                btn.disabled = false;
                btn.className = "w-full py-4 rounded-2xl font-black text-sm bg-green-600 text-white shadow-lg transition-all";
            } else {
                status.innerHTML = `<span class="text-red-500">Insuffisant : Manque ${(total - paid).toLocaleString()} FCFA</span>`;
                btn.disabled = true;
                btn.className = "w-full py-4 rounded-2xl font-black text-sm bg-slate-200 text-slate-400 transition-all";
            }
        }

        function finalizeOrder() {
            const total = updateTotal();
            const tableNum = document.getElementById('table-number').value;
            
            // Validation Table
            if (orderMode === 'Sur Place' && !tableNum) {
                alert("Veuillez entrer le numéro de table.");
                return;
            }

            const ticket = document.getElementById('printable-ticket');
            const orderId = orderCounter++;
            let items = [];
            let itemsHtml = "";
            for (const [id, qty] of Object.entries(cart)) {
                if (qty > 0) {
                    const p = products.find(prod => prod.id == id);
                    items.push({ name: p.name, qty: qty, price: p.price });
                    itemsHtml += `<div class="flex justify-between"><span>${p.name} x${qty}</span> <span>${(p.price * qty).toLocaleString()}</span></div>`;
                }
            }

            saveOrderToFirebase({
                orderId,
                items,
                total,
                mode: orderMode,
                table: tableNum || "N/A"
            });

            ticket.innerHTML = `
                <div class="text-center mb-4">
                    <h2 class="font-black text-2xl">X-PRESS RESTO</h2>
                    <p class="text-[10px]">${orderMode.toUpperCase()} ${orderMode === 'Sur Place' ? ' | TABLE ' + tableNum : ''}</p>
                    <p class="text-[10px]">COMMANDE #${orderId}</p>
                    <p class="text-[10px]">${new Date().toLocaleString()}</p>
                </div>
                <div class="border-t border-b border-dashed border-black py-4 my-2">
                    ${itemsHtml}
                </div>
                <div class="text-right font-black text-lg mb-4">TOTAL: ${total.toLocaleString()} FCFA</div>
            `;
            
            document.getElementById('payment-modal').classList.replace('flex', 'hidden');
            window.print();
            
            // Reset
            cart = {};
            document.getElementById('table-number').value = '';
            renderMenu();
            updateTotal();
        }

        function exportToExcel() {
            let csv = "Produit,Quantite,Prix,Total\n";
            for (const [id, qty] of Object.entries(cart)) {
                if (qty > 0) {
                    const p = products.find(prod => prod.id == id);
                    csv += `${p.name},${qty},${p.price},${p.price * qty}\n`;
                }
            }
            const blob = new Blob([csv], { type: 'text/csv' });
            const url = URL.createObjectURL(blob);
            const a = document.createElement('a');
            a.href = url;
            a.download = `Commande_${new Date().getTime()}.csv`;
            a.click();
        }

        window.onload = () => { renderCategories(); renderMenu(); };
    </script>
</body>
</html>

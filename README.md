<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Checkout - Faven Lightings</title>
    <style>
        body { font-family: 'Segoe UI', system-ui, -apple-system, sans-serif; background: #f4f6f9; padding: 20px; color: #2d3748; display: flex; justify-content: center; margin: 0; }
        .container { width: 100%; max-width: 600px; }
        .card { background: white; padding: 20px; border-radius: 12px; box-shadow: 0 4px 12px rgba(0,0,0,0.03); border: 1px solid #edf2f7; margin-bottom: 20px; }
        h2, h3 { margin-top: 0; color: #1a202c; }
        .btn { background: #3182ce; color: white; border: none; padding: 12px 16px; border-radius: 8px; font-weight: 600; cursor: pointer; font-size: 14px; width: 100%; transition: background 0.2s; }
        .btn:hover { background: #2b6cb0; }
        .btn-location { background: #319795; margin-bottom: 12px; }
        .btn-location:hover { background: #285e61; }
        .item-row { display: flex; justify-content: space-between; align-items: center; padding: 8px 0; border-bottom: 1px solid #edf2f7; font-size: 14px; }
        .summary-total { display: flex; justify-content: space-between; font-weight: 700; font-size: 16px; margin-top: 15px; color: #1a202c; }
        #locationStatusDisplay { font-size: 13px; padding: 10px; border-radius: 8px; background: #edf2f7; color: #4a5568; margin-bottom: 15px; text-align: center; }
    </style>
</head>
<body>

    <div class="container">
        <h2>🛍️ Checkout & Delivery</h2>

        <!-- Order Summary Card -->
        <div class="card">
            <h3>Your Order Summary</h3>
            <div id="cartItemsList">Loading cart...</div>
            
            <div style="margin-top: 15px; font-size: 14px;">
                <div style="display: flex; justify-content: space-between; margin-bottom: 5px;">
                    <span>Items Total:</span>
                    <span id="subtotalText">₹0</span>
                </div>
                <div style="display: flex; justify-content: space-between; margin-bottom: 5px;">
                    <span>Delivery Charge:</span>
                    <span id="deliveryText">₹30</span>
                </div>
            </div>
            
            <div class="summary-total">
                <span>Grand Total:</span>
                <span id="grandTotalText">₹0</span>
            </div>
        </div>

        <!-- Location & Checkout Card -->
        <div class="card">
            <h3>Delivery Location</h3>
            <div id="locationStatusDisplay">Please fetch your live Google Maps location to calculate delivery.</div>
            <button class="btn btn-location" onclick="fetchLiveLocation()">📍 Fetch Live Google Maps Pin</button>

            <label style="font-size: 13px; font-weight: 600; display: block; margin-top: 10px;">Phone Number</label>
            <input type="text" id="customerPhone" placeholder="Enter your phone number" style="width: 100%; padding: 10px; margin: 8px 0 16px 0; border: 1px solid #cbd5e0; border-radius: 6px; box-sizing: border-box; font-size: 14px;">

            <button class="btn" onclick="submitOrder()">🚀 Place Order</button>
        </div>
    </div>

<script>
    const FIREBASE_URL = "https://test-d34cf-default-rtdb.europe-west1.firebasedatabase.app";
    
    // Faven Lightings Shop Coordinates (Byatarayanapura, Bengaluru)
    const SHOP_LAT = 13.0650;
    const SHOP_LNG = 77.5880;

    let cart = [];
    let deliveryCharge = 30; // default fallback
    let verifiedMapsLink = "";

    function calculateDistance(lat1, lon1, lat2, lon2) {
        const R = 6371;
        const dLat = (lat2 - lat1) * Math.PI / 180;
        const dLon = (lon2 - lon1) * Math.PI / 180;
        const a = 
            Math.sin(dLat/2) * Math.sin(dLat/2) +
            Math.cos(lat1 * Math.PI / 180) * Math.cos(lat2 * Math.PI / 180) * 
            Math.sin(dLon/2) * Math.sin(dLon/2);
        const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
        return R * c;
    }

    function loadCart() {
        // Retrieve cart stored from menu page (expects array of {name, price, quantity})
        cart = JSON.parse(localStorage.getItem('cart') || '[]');
        renderSummary();
    }

    function renderSummary() {
        let container = document.getElementById('cartItemsList');
        if (cart.length === 0) {
            container.innerHTML = '<p style="color:#718096; font-size:14px;">Your cart is empty.</p>';
            document.getElementById('subtotalText').innerText = '₹0';
            document.getElementById('deliveryText').innerText = '₹' + deliveryCharge;
            document.getElementById('grandTotalText').innerText = '₹' + deliveryCharge;
            return;
        }

        let html = '';
        let subtotal = 0;
        cart.forEach(item => {
            let itemTotal = item.price * item.quantity;
            subtotal += itemTotal;
            html += `<div class="item-row"><span>${item.name} (x${item.quantity})</span><span>₹${itemTotal}</span></div>`;
        });

        container.innerHTML = html;
        document.getElementById('subtotalText').innerText = '₹' + subtotal;
        document.getElementById('deliveryText').innerText = '₹' + deliveryCharge;
        document.getElementById('grandTotalText').innerText = '₹' + (subtotal + deliveryCharge);
    }

    async function fetchLiveLocation() {
        if (!navigator.geolocation) {
            alert('Geolocation is not supported by your browser.');
            return;
        }

        const statusBox = document.getElementById('locationStatusDisplay');
        statusBox.style.background = '#ebf8ff';
        statusBox.style.color = '#2b6cb0';
        statusBox.innerHTML = '🔄 Fetching your location and syncing admin pricing rules...';

        navigator.geolocation.getCurrentPosition(async (position) => {
            const lat = position.coords.latitude;
            const lng = position.coords.longitude;
            verifiedMapsLink = `https://www.google.com/maps?q=${lat},${lng}`;
            
            const distanceKm = calculateDistance(SHOP_LAT, SHOP_LNG, lat, lng);

            // Fetch live delivery rules from Firebase Admin settings
            let rules = { tier1Dist: 3, tier1Price: 30, tier2Dist: 6, tier2Price: 60, farPrice: 100 };
            try {
                let res = await fetch(`${FIREBASE_URL}/settings/delivery.json`);
                let data = await res.json();
                if (data) rules = data;
            } catch (e) {
                console.log("Could not load rules from Firebase, using defaults.");
            }

            // Apply admin rules
            if (distanceKm <= rules.tier1Dist) {
                deliveryCharge = rules.tier1Price;
            } else if (distanceKm <= rules.tier2Dist) {
                deliveryCharge = rules.tier2Price;
            } else {
                deliveryCharge = rules.farPrice;
            }

            renderSummary();

            statusBox.style.background = '#c6f6d5';
            statusBox.style.color = '#22543d';
            statusBox.innerHTML = `✅ Location Pinned! (${distanceKm.toFixed(1)} km away)<br>Delivery Fee: ₹${deliveryCharge} <br><a href="${verifiedMapsLink}" target="_blank" style="color:#2b6cb0; font-size:12px;">View Map ↗</a>`;
        }, () => {
            statusBox.style.background = '#fed7d7';
            statusBox.style.color = '#9b2c2c';
            statusBox.innerHTML = '❌ Location access denied. Please allow GPS permissions.';
        }, { enableHighAccuracy: true });
    }

    async function submitOrder() {
        if (cart.length === 0) {
            alert('Your cart is empty.');
            return;
        }
        if (!verifiedMapsLink) {
            alert('Please fetch your live Google Maps location pin before placing the order.');
            return;
        }

        let phone = document.getElementById('customerPhone').value.trim();
        if (!phone) {
            alert('Please enter your phone number.');
            return;
        }

        let subtotal = cart.reduce((sum, item) => sum + (item.price * item.quantity), 0);
        let grandTotal = subtotal + deliveryCharge;

        let newOrder = {
            id: 'ORD-' + Math.floor(1000 + Math.random() * 9000),
            phone: phone,
            items: cart,
            deliveryCharge: deliveryCharge,
            grandTotal: grandTotal,
            location: verifiedMapsLink,
            time: new Date().toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })
        };

        try {
            let res = await fetch(`${FIREBASE_URL}/orders.json`);
            let existingOrders = await res.json() || [];
            if (!Array.isArray(existingOrders)) existingOrders = [];

            existingOrders.push(newOrder);

            await fetch(`${FIREBASE_URL}/orders.json`, {
                method: 'PUT',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(existingOrders)
            });

            localStorage.removeItem('cart');
            alert('Order placed successfully!');
            window.location.href = 'https://kshitij-bhuwania.github.io/Menu/'; // Redirect back to menu or success page
        } catch (e) {
            alert('Failed to place order. Check your internet connection.');
        }
    }

    loadCart();
</script>
</body>
</html>

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Secure Checkout</title>
    <style>
        body { font-family: 'Segoe UI', system-ui, -apple-system, sans-serif; background: #f4f6f9; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; box-sizing: border-box; color: #2d3748; margin: 0; }
        .checkout-box { background: white; padding: 30px 20px; border-radius: 16px; width: 100%; max-width: 420px; box-shadow: 0 10px 25px rgba(0,0,0,0.05); border: 1px solid #edf2f7; text-align: center; box-sizing: border-box; }
        h2 { margin-top: 0; color: #1a202c; font-weight: 600; font-size: 22px; }
        .btn { background: #2ed573; color: white; border: none; padding: 12px; border-radius: 8px; width: 100%; font-weight: 600; cursor: pointer; font-size: 14px; margin-top: 12px; transition: background 0.2s; box-sizing: border-box; }
        .btn:hover { background: #26af5f; }
        .btn-map { background: #3182ce; }
        .btn-map:hover { background: #2b6cb0; }
        .btn-back { background: #718096; margin-top: 10px; }
        .btn-back:hover { background: #4a5568; }
        .summary-line { display: flex; justify-content: space-between; margin: 10px 0; font-size: 14px; color: #4a5568; }
        .location-status { background: #e2e8f0; padding: 12px; border-radius: 8px; font-size: 13px; color: #4a5568; margin: 15px 0; word-break: break-all; font-weight: 500; text-align: left; }
    </style>
</head>
<body>

 <div class="checkout-box">
        <h2>Order Checkout</h2>
        <p style="font-size: 13px; color: #718096; margin-bottom: 20px;">We require your live Google Maps location pin to dispatch your order.</p>
        <button class="btn btn-map" onclick="fetchLiveLocation()">📍 Fetch Live Google Maps Pin</button>
      
        <div id="locationStatusDisplay" class="location-status">
            ❌ No location pinned yet. Please click the button above.
        </div>

        <hr style="border: 0; border-top: 1px solid #edf2f7; margin: 20px 0;">
        <div id="checkoutSummary" style="text-align: left;"></div>

        <button class="btn" onclick="placeOrder()">Place Order Now</button>
        <button class="btn btn-back" onclick="returnToMenu()">← Return to Menu</button>
    </div>

<script>
    const FIREBASE_URL = "https://test-d34cf-default-rtdb.europe-west1.firebasedatabase.app";
    
    // Shop Coordinates (Byatarayanapura, Bengaluru)
    const SHOP_LAT = 13.0650;
    const SHOP_LNG = 77.5880;

    const phone = localStorage.getItem('activeCustomerPhone') || 'Customer_' + Math.floor(Math.random() * 9000 + 1000);
    let cartKey = 'cart_' + phone;
    let cart = JSON.parse(localStorage.getItem(cartKey) || '{}');
    let deliveryCharge = 30; // default fallback
    let verifiedMapsLink = '';

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

    function renderSummary() {
        const container = document.getElementById('checkoutSummary');
        let itemTotal = 0;
        let html = '<p style="font-weight:600; color:#1a202c; margin-bottom:12px;">Review Order Summary:</p>';

        let hasItems = false;
        for (let id in cart) {
            hasItems = true;
            let item = cart[id];
            itemTotal += item.price * item.quantity;
            html += `<div class="summary-line"><span>${item.name} (x${item.quantity})</span><span>₹${item.price * item.quantity}</span></div>`;
        }

        if (!hasItems) {
            html += '<p style="color: #e53e3e; font-size: 13px; text-align: center;">Your cart is empty. Add items first.</p>';
        }

        let grandTotal = itemTotal > 0 ? itemTotal + deliveryCharge : 0;
        html += `<hr style="border:0; border-top:1px dashed #cbd5e0; margin:15px 0;">`;
        html += `<div class="summary-line"><span>Item Total:</span><span>₹${itemTotal}</span></div>`;
        html += `<div class="summary-line"><span>Delivery Fee:</span><span>₹${deliveryCharge}</span></div>`;
        html += `<div class="summary-line" style="font-weight:bold; font-size:16px; color:#1a202c;"><span>Grand Total:</span><span>₹${grandTotal}</span></div>`;

        container.innerHTML = html;
    }

    async function fetchLiveLocation() {
        if (!navigator.geolocation) {
            alert('Geolocation is not supported by your browser.');
            return;
        }

        const statusBox = document.getElementById('locationStatusDisplay');
        statusBox.style.background = '#ebf8ff';
        statusBox.style.color = '#2b6cb0';
        statusBox.innerHTML = '🔄 Fetching location & calculating distance delivery fee...';

        navigator.geolocation.getCurrentPosition(async (position) => {
            const lat = position.coords.latitude;
            const lng = position.coords.longitude;
            verifiedMapsLink = `https://www.google.com/maps?q=${lat},${lng}`;
            
            const distanceKm = calculateDistance(SHOP_LAT, SHOP_LNG, lat, lng);

            // Fetch live delivery rules configured in Admin Panel
            let rules = { baseKm: 3, basePrice: 30, extraPricePerKm: 15 };
            try {
                let res = await fetch(`${FIREBASE_URL}/settings/delivery.json`);
                let data = await res.json();
                if (data) rules = data;
            } catch (e) {
                console.log("Could not load rules from Firebase, using defaults.");
            }

            // Calculate distance-based delivery fee
            if (distanceKm <= rules.baseKm) {
                deliveryCharge = rules.basePrice;
            } else {
                let extraKm = distanceKm - rules.baseKm;
                deliveryCharge = rules.basePrice + Math.ceil(extraKm * rules.extraPricePerKm);
            }

            renderSummary();

            statusBox.style.background = '#c6f6d5';
            statusBox.style.color = '#22543d';
            statusBox.innerHTML = `✅ Location Pinned (${distanceKm.toFixed(1)} km away)!<br><a href="${verifiedMapsLink}" target="_blank" style="color:#2b6cb0; font-size:12px;">View Map ↗</a>`;
        }, () => {
            statusBox.style.background = '#fed7d7';
            statusBox.style.color = '#9b2c2c';
            statusBox.innerHTML = '❌ Unable to retrieve location. Please allow GPS permissions.';
        }, { enableHighAccuracy: true });
    }

    async function placeOrder() {
        if (!verifiedMapsLink) {
            alert('Error: You must fetch your Google Maps location pin before checking out!');
            return;
        }

        let itemTotal = Object.values(cart).reduce((sum, i) => sum + (i.price * i.quantity), 0);
        if (itemTotal === 0) {
            alert('Your cart is empty!');
            return;
        }

        let newOrder = {
            id: 'ORD-' + Math.floor(Math.random() * 90000 + 10000),
            phone: phone,
            location: verifiedMapsLink,
            items: Object.values(cart),
            itemTotal: itemTotal,
            deliveryCharge: deliveryCharge,
            grandTotal: itemTotal + deliveryCharge,
            time: new Date().toLocaleTimeString()
        };

        try {
            let res = await fetch(`${FIREBASE_URL}/orders.json`);
            let orders = await res.json();
            if (!Array.isArray(orders)) orders = [];

            orders.unshift(newOrder);

            await fetch(`${FIREBASE_URL}/orders.json`, {
                method: 'PUT',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(orders)
            });

            localStorage.removeItem(cartKey);
            alert('Order placed successfully!');
            window.location.href = 'https://kshitij-bhuwania.github.io/Menu/';
        } catch (e) {
            alert('Network error placing order. Please check your connection.');
        }
    }

    function returnToMenu() {
        window.location.href = 'https://kshitij-bhuwania.github.io/Menu/';
    }

    renderSummary();
</script>
</body>
</html>

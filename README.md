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
    // Verify customer is logged in; if not, bounce back to login
    const phone = localStorage.getItem('activeCustomerPhone');
    if (!phone) {
        window.location.href = 'login.html';
    }

    // Isolate cart uniquely to this specific logged-in customer's phone number
    let cartKey = 'cart_' + phone;
    let cart = JSON.parse(localStorage.getItem(cartKey) || '{}');
    let deliveryCharge = parseInt(localStorage.getItem('deliveryCharge') || '40');
    let verifiedMapsLink = '';

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
            html += '<p style="color: #e53e3e; font-size: 13px; text-align: center;">Your cart is currently empty.</p>';
        }

        let grandTotal = itemTotal > 0 ? itemTotal + deliveryCharge : 0;
        html += `<hr style="border:0; border-top:1px dashed #cbd5e0; margin:15px 0;">`;
        html += `<div class="summary-line"><span>Item Total:</span><span>₹${itemTotal}</span></div>`;
        html += `<div class="summary-line"><span>Delivery Fee:</span><span>₹${deliveryCharge}</span></div>`;
        html += `<div class="summary-line" style="font-weight:bold; font-size:16px; color:#1a202c;"><span>Grand Total:</span><span>₹${grandTotal}</span></div>`;

        container.innerHTML = html;
    }

    function fetchLiveLocation() {
        if (!navigator.geolocation) {
            alert('Geolocation is not supported by your browser.');
            return;
        }
        navigator.geolocation.getCurrentPosition((position) => {
            const lat = position.coords.latitude;
            const lng = position.coords.longitude;
            verifiedMapsLink = `https://www.google.com/maps?q=${lat},${lng}`;
            
            const statusBox = document.getElementById('locationStatusDisplay');
            statusBox.style.background = '#c6f6d5';
            statusBox.style.color = '#22543d';
            statusBox.innerHTML = `✅ Location Pinned Successfully!<br><a href="${verifiedMapsLink}" target="_blank" style="color:#2b6cb0; font-size:12px;">View Coordinates on Map ↗</a>`;
        }, () => {
            alert('Unable to retrieve location. Please allow GPS permissions in your browser.');
        });
    }

    function placeOrder() {
        if (!verifiedMapsLink) {
            alert('Error: You must fetch your Google Maps location pin before you can check out!');
            return;
        }

        let itemTotal = Object.values(cart).reduce((sum, i) => sum + (i.price * i.quantity), 0);
        if (itemTotal === 0) {
            alert('Your cart is empty! Please add items first.');
            return;
        }

        let orders = JSON.parse(localStorage.getItem('liveOrders') || '[]');
        let serialNumber = orders.length + 1;

        let newOrder = {
            serialNumber: serialNumber,
            phone: phone, // Captures exact customer phone so orders never mix up
            location: verifiedMapsLink,
            items: Object.values(cart),
            totalItemsCount: Object.values(cart).reduce((sum, i) => sum + i.quantity, 0),
            itemTotal: itemTotal,
            deliveryCharge: deliveryCharge,
            grandTotal: itemTotal + deliveryCharge,
            timestamp: new Date().toLocaleTimeString()
        };

        orders.unshift(newOrder);
        localStorage.setItem('liveOrders', JSON.stringify(orders));
        
        // Clear only this specific customer's cart after successful order placement
        localStorage.removeItem(cartKey);

        alert('Order placed successfully!');
        // Redirect back to menu or kitchen order view link
        window.location.href = 'https://kshitij-bhuwania.github.io/Menu/';
    }

    function returnToMenu() {
        // Directs back to your hosted menu link while preserving their active customer login session
        window.location.href = 'https://kshitij-bhuwania.github.io/Menu/';
    }

    renderSummary();
</script>
</body>
</html>

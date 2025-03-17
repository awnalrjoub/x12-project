<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>X12 Token Purchase</title>
  <link rel="stylesheet" href="styles.css">
</head>
<body>
  <div class="container">
    <h1>X12 Token Purchase</h1>

    <!-- Wallet Connection -->
    <button id="connect-wallet-btn">Connect Wallet</button>
    <p id="wallet-status">Wallet not connected</p>

    <!-- Payment Section -->
    <button id="pay-stripe-btn" disabled>Pay with Stripe</button>
    <button id="pay-solana-btn" disabled>Pay with Solana</button>
    <p id="payment-message"></p>
  </div>

  <script src="https://js.stripe.com/v3/"></script>
  <script src="app.js"></script>
</body>
</html>
/* General Styles */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Arial', sans-serif;
  background-color: #f0f4f8;
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
}

.container {
  text-align: center;
  background-color: #fff;
  padding: 40px;
  border-radius: 10px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
  width: 300px;
}

h1 {
  font-size: 24px;
  margin-bottom: 20px;
  color: #333;
}

button {
  background-color: #4CAF50;
  color: white;
  border: none;
  padding: 10px 20px;
  font-size: 16px;
  border-radius: 5px;
  cursor: pointer;
  margin: 10px 0;
  width: 100%;
}

button:disabled {
  background-color: #ddd;
  cursor: not-allowed;
}

button:hover:not(:disabled) {
  background-color: #45a049;
}

p {
  color: #333;
  font-size: 14px;
  margin-top: 10px;
}

#payment-message {
  font-size: 14px;
  color: blue;
}
document.addEventListener('DOMContentLoaded', () => {
  const connectBtn = document.getElementById('connect-wallet-btn');
  const payStripeBtn = document.getElementById('pay-stripe-btn');
  const paySolanaBtn = document.getElementById('pay-solana-btn');
  const walletStatus = document.getElementById('wallet-status');
  const paymentMessage = document.getElementById('payment-message');

  let userWalletAddress = null;
  let isProcessing = false; // Prevent multiple clicks

  // Initialize Stripe with the public key (Replace with your key)
  const stripe = Stripe('YOUR_STRIPE_PUBLISHABLE_KEY');

  // Check if Phantom Wallet is installed
  const isPhantomInstalled = () => window.solana?.isPhantom;

  // Update wallet status in the UI
  function updateWalletStatus(address) {
    if (address) {
      const displayAddress = `${address.slice(0, 4)}...${address.slice(-4)}`;
      walletStatus.textContent = `Connected: ${displayAddress}`;
      payStripeBtn.disabled = false;
      paySolanaBtn.disabled = false;
    } else {
      walletStatus.textContent = 'Wallet not connected';
      payStripeBtn.disabled = true;
      paySolanaBtn.disabled = true;
    }
  }

  // Function to toggle loading indicator
  function toggleLoading(button, isLoading, defaultText) {
    button.disabled = isLoading;
    button.textContent = isLoading ? 'Processing...' : defaultText;
  }

  // Connect to Phantom Wallet
  async function connectWallet() {
    if (!isPhantomInstalled()) {
      alert('Phantom Wallet not detected. Please install it first.');
      return;
    }

    toggleLoading(connectBtn, true, 'Connecting...');

    try {
      const response = await window.solana.connect();
      userWalletAddress = response.publicKey.toString();
      updateWalletStatus(userWalletAddress);
    } catch (err) {
      console.error('Failed to connect to wallet:', err);
      alert('Failed to connect. Please try again.');
    } finally {
      toggleLoading(connectBtn, false, 'Connect Wallet');
    }
  }

  // Handle token purchase via Stripe
  async function purchaseTokensWithStripe() {
    if (!userWalletAddress) {
      alert('Please connect your Solana wallet first.');
      return;
    }
    if (isProcessing) return; // Prevent multiple clicks
    isProcessing = true;

    toggleLoading(payStripeBtn, true, 'Processing payment...');
    paymentMessage.textContent = 'Processing payment via Stripe...';
    paymentMessage.style.color = 'blue';

    try {
      const response = await fetch('/create-checkout-session', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ walletAddress: userWalletAddress })
      });

      const session = await response.json();
      if (session.error) throw new Error(session.error);

      const { error } = await stripe.redirectToCheckout({ sessionId: session.id });
      if (error) throw error;
    } catch (err) {
      console.error('Stripe payment failed:', err);
      paymentMessage.textContent = 'Payment error: ' + err.message;
      paymentMessage.style.color = 'red';
    } finally {
      toggleLoading(payStripeBtn, false, 'Pay with Stripe');
      isProcessing = false;
    }
  }

  // Handle token purchase via Solana
  async function purchaseTokensWithSolana() {
    if (!userWalletAddress) {
      alert('Please connect your Solana wallet first.');
      return;
    }
    if (isProcessing) return; // Prevent multiple clicks
    isProcessing = true;

    toggleLoading(paySolanaBtn, true, 'Processing payment...');
    paymentMessage.textContent = 'Processing payment via Solana...';
    paymentMessage.style.color = 'blue';

    try {
      // Check if Phantom Wallet is connected
      const wallet = window.solana;
      if (!wallet) {
        alert('Phantom Wallet not connected!');
        return;
      }

      // Create the transaction on the Solana network
      const connection = new Connection("https://api.mainnet-beta.solana.com", "confirmed");
      const transaction = new Transaction().add(
        SystemProgram.transfer({
          fromPubkey: wallet.publicKey, // User's public key
          toPubkey: new PublicKey('TARGET_WALLET_ADDRESS'), // Replace with your receiving wallet address
          lamports: 1000000000, // 1 SOL = 1 billion lamports (adjust amount as needed)
        })
      );

      // Sign and send the transaction
      const signature = await wallet.signAndSendTransaction(transaction);
      await connection.confirmTransaction(signature);

      paymentMessage.textContent = 'Payment successful via Solana!';
    } catch (err) {
      console.error('Solana payment failed:', err);
      paymentMessage.textContent = 'Payment error: ' + err.message;
      paymentMessage.style.color = 'red';
    } finally {
      toggleLoading(paySolanaBtn, false, 'Pay with Solana');
      isProcessing = false;
    }
  }

  // Listen for wallet disconnection event
  window.solana?.on('disconnect', () => {
    userWalletAddress = null;
    updateWalletStatus(null);
    alert('Wallet disconnected.');
  });

  // Attach event listeners
  connectBtn.addEventListener('click', connectWallet);
  payStripeBtn.addEventListener('click', purchaseTokensWithStripe);
  paySolanaBtn.addEventListener('click', purchaseTokensWithSolana);

  // Set initial wallet status
  updateWalletStatus(null);
});

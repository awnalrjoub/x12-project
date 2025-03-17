document.addEventListener('DOMContentLoaded', () => {
  const connectBtn = document.getElementById('connect-wallet-btn');
  const buyBtn = document.getElementById('buy-btn');
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
      buyBtn.disabled = false;
    } else {
      walletStatus.textContent = 'Wallet not connected';
      buyBtn.disabled = true;
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
  async function purchaseTokens() {
    if (!userWalletAddress) {
      alert('Please connect your Solana wallet first.');
      return;
    }
    if (isProcessing) return; // Prevent multiple clicks
    isProcessing = true;

    toggleLoading(buyBtn, true, 'Processing payment...');
    paymentMessage.textContent = 'Processing payment...';
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
      console.error('Payment failed:', err);
      paymentMessage.textContent = 'Payment error: ' + err.message;
      paymentMessage.style.color = 'red';
    } finally {
      toggleLoading(buyBtn, false, 'Buy Tokens');
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
  buyBtn.addEventListener('click', purchaseTokens);

  // Set initial wallet status
  updateWalletStatus(null);
});

# X12 Meme Coin Project
document.addEventListener('DOMContentLoaded', () => {
  const connectBtn = document.getElementById('connect-wallet-btn');
  const buyBtn = document.getElementById('buy-btn');
  const walletStatus = document.getElementById('wallet-status');
  const paymentMessage = document.getElementById('payment-message');

  let userWalletAddress = null;
  let isProcessing = false; // لمنع النقر المتكرر

  // تهيئة Stripe بالمفتاح العام (استبدله بمفتاحك)
  const stripe = Stripe('YOUR_STRIPE_PUBLISHABLE_KEY');

  // التحقق من وجود محفظة Phantom
  const isPhantomInstalled = () => window.solana?.isPhantom;

  // تحديث حالة المحفظة في الواجهة
  function updateWalletStatus(address) {
    if (address) {
      const displayAddress = ${address.slice(0, 4)}...${address.slice(-4)};
      walletStatus.textContent = متصل: ${displayAddress};
      buyBtn.disabled = false;
    } else {
      walletStatus.textContent = 'لم يتم الاتصال بالمحفظة';
      buyBtn.disabled = true;
    }
  }

  // وظيفة لعرض وإخفاء مؤشر التحميل (Loader)
  function toggleLoading(button, isLoading, defaultText) {
    button.disabled = isLoading;
    button.textContent = isLoading ? 'جارٍ المعالجة...' : defaultText;
  }

  // الاتصال بمحفظة Phantom
  async function connectWallet() {
    if (!isPhantomInstalled()) {
      alert('لم يتم العثور على محفظة Phantom. الرجاء تثبيتها أولًا.');
      return;
    }

    toggleLoading(connectBtn, true, 'اتصل بالمحفظة');

    try {
      const response = await window.solana.connect();
      userWalletAddress = response.publicKey.toString();
      updateWalletStatus(userWalletAddress);
    } catch (err) {
      console.error('فشل الاتصال بالمحفظة:', err);
      alert('فشل الاتصال بالمحفظة. الرجاء المحاولة مرة أخرى.');
    } finally {
      toggleLoading(connectBtn, false, 'اتصل بالمحفظة');
    }
  }

  // تنفيذ عملية الشراء عبر Stripe
  async function purchaseTokens() {
    if (!userWalletAddress) {
      alert('الرجاء الاتصال بمحفظة Solana أولاً.');
      return;
    }
    if (isProcessing) return; // منع النقر المتكرر
    isProcessing = true;

    toggleLoading(buyBtn, true, 'اشترِ التوكنات');
    paymentMessage.textContent = 'جارٍ معالجة الدفع...';
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
      console.error('فشل الدفع:', err);
      paymentMessage.textContent = 'حدث خطأ أثناء الدفع: ' + err.message;
      paymentMessage.style.color = 'red';
    } finally {
      toggleLoading(buyBtn, false, 'اشترِ التوكنات');
      isProcessing = false;
    }
  }

  // الاستماع إلى حدث فصل المحفظة
  window.solana?.on('disconnect', () => {
    userWalletAddress = null;
    updateWalletStatus(null);
    alert('تم قطع الاتصال بالمحفظة.');
  });

  // ربط الأزرار بالوظائف
  connectBtn.addEventListener('click', connectWallet);
  buyBtn.addEventListener('click', purchaseTokens);

  // ضبط الحالة الافتراضية
  updateWalletStatus(null);
})

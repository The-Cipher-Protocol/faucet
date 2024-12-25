<script>
  import { onMount } from 'svelte';
  import { getAddress } from '@ethersproject/address';
  import { CloudflareProvider } from '@ethersproject/providers';
  import { setDefaults as setToast, toast } from 'bulma-toast';

  let input = null;
  let faucetInfo = {
    account: '0x0000000000000000000000000000000000000000',
    network: 'Ciper TESTNET',
    payout: 2,
    symbol: 'Ciper',
    hcaptcha_sitekey: '',
  };

  let mounted = false;
  let hcaptchaLoaded = false;

  onMount(async () => {
    const res = await fetch('/api/info');
    faucetInfo = await res.json();
    mounted = true;
    initMouseEffect();
  });

  window.hcaptchaOnLoad = () => {
    hcaptchaLoaded = true;
  };

  $: document.title = `${faucetInfo.symbol} ${capitalize(
    faucetInfo.network,
  )} Faucet`;

  let widgetID;
  $: if (mounted && hcaptchaLoaded) {
    widgetID = window.hcaptcha.render('hcaptcha', {
      sitekey: faucetInfo.hcaptcha_sitekey,
    });
  }

  setToast({
    position: 'bottom-center',
    dismissible: true,
    pauseOnHover: true,
    closeOnClick: false,
    animate: { in: 'fadeIn', out: 'fadeOut' },
  });

  async function handleRequest() {
    let address = input;
    if (address === null) {
      toast({ message: 'Input required', type: 'is-warning' });
      return;
    }

    if (address.endsWith('.eth')) {
      try {
        const provider = new CloudflareProvider();
        address = await provider.resolveName(address);
        if (!address) {
          toast({ message: 'Invalid ENS name', type: 'is-warning' });
          return;
        }
      } catch (error) {
        toast({ message: error.reason, type: 'is-warning' });
        return;
      }
    }

    try {
      address = getAddress(address);
    } catch (error) {
      toast({ message: error.reason, type: 'is-warning' });
      return;
    }

    try {
      let headers = {
        'Content-Type': 'application/json',
      };

      if (hcaptchaLoaded) {
        const { response } = await window.hcaptcha.execute(widgetID, {
          async: true,
        });
        headers['h-captcha-response'] = response;
      }

      const res = await fetch('/api/claim', {
        method: 'POST',
        headers,
        body: JSON.stringify({
          address,
        }),
      });

      let { msg } = await res.json();
      let type = res.ok ? 'is-success' : 'is-warning';
      toast({ message: msg, type });
    } catch (err) {
      console.error(err);
    }
  }

  function capitalize(str) {
    const lower = str.toLowerCase();
    return str.charAt(0).toUpperCase() + lower.slice(1);
  }

  function initMouseEffect() {
    const particlesContainer = document.createElement('div');
    particlesContainer.className = 'particles';
    document.body.appendChild(particlesContainer);

    function createParticle(x, y) {
      const particle = document.createElement('div');
      particle.className = 'particle';
      particle.style.left = `${x}px`;
      particle.style.top = `${y}px`;
      particlesContainer.appendChild(particle);

      setTimeout(() => {
        particle.remove();
      }, 1000);
    }

    document.addEventListener('mousemove', (e) => {
      createParticle(e.pageX, e.pageY);
    });
  }
</script>

<svelte:head>
  {#if mounted && faucetInfo.hcaptcha_sitekey}
    <script
      src="https://hcaptcha.com/1/api.js?onload=hcaptchaOnLoad&render=explicit"
      async
      defer
    ></script>
  {/if}
</svelte:head>

<main>
  <section class="hero is-fullheight">
    <div class="hero-head">
      <nav class="navbar">
        <div class="container">
          <div class="navbar-brand">
            <a class="navbar-item" href="../..">
              <span class="icon">
                <i class="fa fa-tint" />
              </span>
              <span><b>{faucetInfo.symbol} Faucet</b></span>
            </a>
          </div>
          <div id="navbarMenu" class="navbar-menu">
            <div class="navbar-end">
              <span class="navbar-item">
                <a
                  class="button is-light is-outlined"
                  href="https://www.cipherprotocol.io/"
                >
                  <span class="icon">
                    <i class="fa fa-info" />
                  </span>
                  <span>Visit Cipher Protocol</span>
                </a>
              </span>
            </div>
          </div>
        </div>
      </nav>
    </div>

    <div class="hero-body">
      <div class="container has-text-centered">
        <div class="column is-6 is-offset-3">
          <h1 class="title has-text-white">
            Receive {faucetInfo.payout}
            {faucetInfo.symbol} per request
          </h1>
          <h2 class="subtitle has-text-grey-light">
            Serving from {faucetInfo.account}
          </h2>
          <div id="hcaptcha" data-size="invisible"></div>
          <div class="box has-background-dark has-text-light">
            <div class="field is-grouped">
              <p class="control is-expanded">
                <input
                  bind:value={input}
                  class="input is-rounded"
                  type="text"
                  placeholder="Enter your address"
                />
              </p>
              <p class="control">
                <button
                  on:click={handleRequest}
                  class="button is-primary is-rounded"
                >
                  Request
                </button>
              </p>
            </div>
          </div>
        </div>
      </div>
    </div>
  </section>
</main>

<style>
  :root {
    --primary-blue: #4169E1;
    --secondary-blue: #1E90FF;
    --dark-blue: #000B2E;
    --light-blue: #87CEFA;
    --hover-blue: #6495ED;
  }

  .hero {
    background: linear-gradient(135deg, var(--dark-blue), #000B2E);
    color: #FFFFFF;
    position: relative;
    overflow: hidden;
  }

  .title {
    font-size: 2.5rem;
    font-weight: bold;
    color: var(--light-blue);
    text-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
  }

  .subtitle {
    font-size: 1.5rem;
    margin-bottom: 2rem;
    color: #B0C4DE;
  }

  .box {
    background: rgba(13, 20, 43, 0.7) !important;
    border-radius: 12px;
    padding: 2rem;
    box-shadow: 0 8px 16px rgba(0, 0, 0, 0.3);
    border: 1px solid rgba(65, 105, 225, 0.2);
  }

  .input {
    background: rgba(255, 255, 255, 0.05);
    border: 1px solid var(--primary-blue);
    padding: 0.75rem;
    font-size: 1.1rem;
    border-radius: 8px;
    color: white;
    transition: all 0.3s ease;
  }

  .input:focus {
    border-color: var(--secondary-blue);
    box-shadow: 0 0 0 2px rgba(65, 105, 225, 0.2);
  }

  .input::placeholder {
    color: rgba(255, 255, 255, 0.5);
  }

  .button.is-primary {
    background-color: var(--primary-blue);
    color: white;
    font-size: 1.1rem;
    padding: 0.75rem 1.5rem;
    border-radius: 8px;
    border: none;
    transition: all 0.3s ease;
  }

  .button.is-primary:hover {
    background-color: var(--hover-blue);
    transform: translateY(-1px);
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
  }

  .navbar-item span {
    color: var(--light-blue);
  }

  .button.is-light.is-outlined {
    border-color: var(--primary-blue);
    color: var(--light-blue);
    background: transparent;
    transition: all 0.3s ease;
  }

  .button.is-light.is-outlined:hover {
    background-color: rgba(65, 105, 225, 0.1);
    border-color: var(--secondary-blue);
  }

  .particles {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
    z-index: 0;
  }

  .particle {
    position: absolute;
    width: 8px;
    height: 8px;
    background-color: var(--light-blue);
    border-radius: 50%;
    pointer-events: none;
    transform: scale(0);
    opacity: 0.6;
    animation: particleAnimation 1s ease-out forwards;
  }

  @keyframes particleAnimation {
    0% {
      transform: scale(0);
      opacity: 0.6;
    }
    50% {
      transform: scale(1);
      opacity: 0.3;
    }
    100% {
      transform: scale(0);
      opacity: 0;
    }
  }
</style>

+++
title = "Certificates"
layout = "simple"
+++

<style>
  .cert-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
    gap: 2rem;
    margin: 2rem 0;
    perspective: 1000px;
  }

  .cert-card {
    aspect-ratio: 4 / 3;
    cursor: pointer;
    position: relative;
    transform-style: preserve-3d;
    transition: transform 0.6s ease;
  }

  .cert-card.flipped {
    transform: rotateY(180deg);
  }

  .cert-face {
    position: absolute;
    inset: 0;
    border-radius: 12px;
    backface-visibility: hidden;
    -webkit-backface-visibility: hidden;
    display: flex;
    align-items: center;
    justify-content: center;
    border: 1px solid rgba(255, 255, 255, 0.1);
    overflow: hidden;
  }

  .cert-front {
    background: rgba(255, 255, 255, 0.03);
    padding: 2rem;
    text-align: center;
  }

  .cert-front h3 {
    margin: 0;
    font-size: 1.35rem;
    font-weight: 700;
    line-height: 1.4;
  }

  .cert-hint {
    position: absolute;
    bottom: 1rem;
    font-size: 0.75rem;
    opacity: 0.4;
  }

  .cert-back {
    transform: rotateY(180deg);
    background: rgba(0, 0, 0, 0.1);
  }

  .cert-back img {
    width: 100%;
    height: 100%;
    object-fit: contain;
    padding: 0.5rem;
  }
</style>

<div class="cert-grid">
  <div class="cert-card" onclick="this.classList.toggle('flipped')">
    <div class="cert-face cert-front">
      <div>
        <h3>Cyber Game Kenya 2026</h3>
        <span class="cert-hint">click to flip</span>
      </div>
    </div>
    <div class="cert-face cert-back">
      <img src="/img/certs/cyber-game-kenya-2026.png" alt="Cyber Game Kenya 2026" loading="lazy">
    </div>
  </div>

  <div class="cert-card" onclick="this.classList.toggle('flipped')">
    <div class="cert-face cert-front">
      <div>
        <h3>HKCERT 2025</h3>
        <span class="cert-hint">click to flip</span>
      </div>
    </div>
    <div class="cert-face cert-back">
      <img src="/img/certs/hkcert-2025.png" alt="HKCERT 2025" loading="lazy">
    </div>
  </div>
</div>

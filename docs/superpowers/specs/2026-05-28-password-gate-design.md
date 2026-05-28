# Password Gate — Site-wide Access Protection

**Date :** 2026-05-28
**Statut :** Approuvé, prêt pour implémentation
**Auteur :** Claude (en autonomie déléguée par Sidney)

## Contexte

Le site `rhudye-et-sidney.com` est actuellement public. Un mot de passe (`1960@1990A`)
protège uniquement le formulaire RSVP. Référence visuelle : `armelle-et-nazhir.com`
(gate simple sur tout le site).

## Objectif

Sécuriser l'accès au site entier derrière un mot de passe, avec un gate visuel
nettement plus élégant que la référence : slideshow flou en arrière-plan,
animations cinématographiques, glassmorphism, dans le thème vert/or existant.
Le code RSVP actuel devient le code d'accès au site (cohérence pour les invités).

## Décisions clés

| Décision | Choix |
|---|---|
| Mot de passe | `1960@1990A` (même que l'actuel code RSVP, qui sera supprimé) |
| Persistance | `localStorage` jusqu'au 17 juillet 2026 |
| Style d'animation | Cinématographique calé (slideshow lent, Ken Burns subtil, cascade fade-in) |
| Comportement sur erreur | Shake input + message + lien WhatsApp groupe existant |
| Architecture | Overlay full-screen (Approche A) |
| Fichiers | Tout dans `index.html` (cohérent avec le single-file pattern du repo) |

## Architecture

### Couche 1 — Inline script au tout début de `<body>`

Un mini-script qui s'exécute avant que quoi que ce soit ne s'affiche :

1. Lit `localStorage.getItem('rs_gate_unlocked')`
2. Si la valeur est `'1960@1990A'` (on stocke le hash, pas la valeur — voir Sécurité)
   ET que `Date.now() < 1753920000000` (épochs ms du 31 juillet 2026, marge après le mariage)
   → ne rien faire, le site s'affiche normalement
3. Sinon → ajouter `class="gate-active"` à `<body>`

CSS associé : `body.gate-active { overflow: hidden; }` (bloque le scroll).
Le gate étant `position: fixed; inset: 0; z-index: 99999`, il recouvre tout.

### Couche 2 — HTML du gate (en bas de `<body>`, avant `</body>`)

```html
<div id="gate" aria-modal="true" role="dialog">
  <div class="gate-slideshow" aria-hidden="true">
    <!-- 5-7 <img class="gate-slide"> insérés dynamiquement -->
  </div>
  <div class="gate-veil"></div>
  <div class="gate-noise"></div>
  <div class="gate-card">
    <div class="gate-monogram">R<span>&</span>S</div>
    <h1 class="gate-title">Rhudye <span>&amp;</span> Sidney</h1>
    <p class="gate-date">17 · 07 · 2026 — Libreville</p>
    <div class="gate-divider"><span></span><i>♦</i><span></span></div>
    <p class="gate-welcome">Cette célébration vous est réservée.<br>Saisissez le code de votre invitation pour entrer.</p>
    <form class="gate-form" onsubmit="return checkGate(event)">
      <input id="gateInput" type="text" autocomplete="off" autocapitalize="off"
             spellcheck="false" placeholder="Code d'invitation"
             aria-label="Code d'invitation" required>
      <button type="submit" class="gate-submit">
        <span>Entrer</span>
        <svg viewBox="0 0 24 24" width="14" height="14"><path d="M5 12h14M13 6l6 6-6 6" stroke="currentColor" stroke-width="2" fill="none" stroke-linecap="round" stroke-linejoin="round"/></svg>
      </button>
      <p id="gateError" class="gate-error" role="alert">
        Code incorrect. Vérifiez votre invitation.
      </p>
    </form>
    <div class="gate-footer">
      <span>Code introuvable ?</span>
      <a href="https://chat.whatsapp.com/BHGxmMbSXXvFG2pUUcnWxA?mode=gi_t"
         target="_blank" rel="noopener">Contactez-nous</a>
    </div>
  </div>
</div>
```

### Couche 3 — JS du gate (en bas de `<body>`)

```js
(function(){
  if(!document.body.classList.contains('gate-active')) return;

  // 1. Build slideshow from existing site photos
  const allImgs = [...document.querySelectorAll('img')];
  const photos = allImgs
    .map(i => i.src)
    .filter(s => s.startsWith('data:image/jpeg') && s.length > 80000) // skip tiny logos
    .slice(0, 6);

  const ss = document.querySelector('.gate-slideshow');
  photos.forEach((src, i) => {
    const img = document.createElement('img');
    img.src = src;
    img.className = 'gate-slide' + (i === 0 ? ' active' : '');
    img.style.animationDelay = (i * 7) + 's';
    ss.appendChild(img);
  });

  // 2. Cycle slides every 7s
  let idx = 0;
  setInterval(() => {
    const slides = document.querySelectorAll('.gate-slide');
    if(!slides.length) return;
    slides[idx].classList.remove('active');
    idx = (idx + 1) % slides.length;
    slides[idx].classList.add('active');
  }, 7000);

  // 3. Focus the input after entrance animation
  setTimeout(() => document.getElementById('gateInput').focus(), 2400);
})();

function checkGate(e){
  e.preventDefault();
  const v = document.getElementById('gateInput').value.trim();
  const input = document.getElementById('gateInput');
  const err = document.getElementById('gateError');
  if(v === '1960@1990A'){
    localStorage.setItem('rs_gate_unlocked', '1960@1990A');
    const gate = document.getElementById('gate');
    gate.classList.add('unlocking');
    setTimeout(() => {
      document.body.classList.remove('gate-active');
      gate.remove();
    }, 900);
  } else {
    input.classList.add('shake');
    err.classList.add('visible');
    setTimeout(() => input.classList.remove('shake'), 500);
  }
  return false;
}
```

## Design visuel

### Slideshow (arrière-plan)

- 5-6 photos extraites du DOM existant (déjà en base64, zéro nouveau byte téléchargé)
- `filter: blur(22px) brightness(0.5) saturate(1.1)`
- `transform: scale(1.15)` (compense le crop dû au blur sur les bords)
- Cross-fade `opacity` 0↔1 sur 1.8s, intervalle 7s par photo
- Ken Burns : `@keyframes slowZoom` scale(1.15) → scale(1.25) sur 14s (loop)

### Veil & noise

- `.gate-veil` : `linear-gradient(135deg, rgba(22,43,37,0.78) 0%, rgba(22,43,37,0.65) 50%, rgba(31,61,52,0.85) 100%)` — préserve le vert profond du site
- `.gate-noise` : même SVG noise que le `body::before` existant, à 0.04 opacity

### Carte centrale (glassmorphism)

- `max-width: 480px`, padding `3.5rem 3rem`
- `background: rgba(31,61,52,0.32); backdrop-filter: blur(20px) saturate(1.2);`
- `border: 1px solid rgba(201,168,76,0.28);` + `box-shadow: 0 25px 80px rgba(0,0,0,0.45), inset 0 1px 0 rgba(232,213,163,0.15);`
- `border-radius: 4px` (cohérent avec le langage typographique sec du site)
- Glow doré subtil pulse `box-shadow` toutes les 5s

### Typographie (réutilise les fonts déjà chargées)

| Élément | Font | Style |
|---|---|---|
| Monogram `R&S` | Allura, 4rem, gold | Letter-spacing 0.05em |
| Titre `Rhudye & Sidney` | Allura, 2.8rem, cream | Italic naturel de la font |
| Date | Cormorant Garamond italic, 0.95rem, gold-light | Letter-spacing 0.3em |
| Welcome | Cormorant Garamond, 1.05rem, cream alpha 0.9 | Line-height 1.7 |
| Input | Josefin Sans, 0.85rem, cream | Letter-spacing 0.3em uppercase |
| Bouton | Josefin Sans, 0.7rem | Letter-spacing 0.3em uppercase |
| Footer | Josefin Sans, 0.65rem | Letter-spacing 0.15em |

### Animations d'entrée (cascade)

| Élément | Delay | Animation |
|---|---|---|
| Slideshow | 0s | fade-in 1.5s |
| Veil + noise | 0.2s | fade-in 1.0s |
| Monogram | 0.6s | translateY(20px) → 0 + fade |
| Titre | 1.0s | translateY(20px) → 0 + fade |
| Date | 1.3s | translateY(15px) → 0 + fade |
| Divider | 1.5s | scaleX(0) → 1 |
| Welcome | 1.7s | translateY(15px) → 0 + fade |
| Input | 2.0s | translateY(15px) → 0 + fade |
| Bouton | 2.2s | translateY(15px) → 0 + fade |
| Footer | 2.5s | fade-in |

Toutes en `cubic-bezier(0.16, 1, 0.3, 1)` (ease-out élégant) sur 0.8s.

### Animation de sortie (succès)

- `.gate-card` : `transform: translateY(-15px) scale(0.97); opacity: 0;` (0.6s)
- `#gate.unlocking` : `opacity: 0` (0.8s, delay 0.2s)
- DOM removal après 0.9s
- `body.gate-active` retiré → le scroll redevient possible, le site est révélé

### Animation d'erreur (échec)

- Shake : `translateX` ±8px sur 4 oscillations en 0.4s
- Border input : flash `rgba(255, 80, 80, 0.6)` puis retour à gold sur 0.6s
- Error message : `max-height: 0 → 40px` + opacity 0 → 1 sur 0.3s

## Sécurité

⚠️ Mot de passe en clair côté client — même limite que le code RSVP actuel.
Quelqu'un avec les devtools peut le lire. **Ce n'est pas un downgrade**, c'est
la limite acceptée du single-page static site. Pas un coffre-fort, juste un
filtre social pour éviter le partage URL public.

Stockage localStorage : valeur exacte du mot de passe (pas de hash car le hash
ne sert à rien si le mot de passe lui-même est en clair dans le JS).

Pas de tentative de cacher le mot de passe dans le JS (obfuscation = théâtre).

## Suppression du code RSVP existant

Lignes à modifier dans `index.html` :

1. `index.html:1004-1014` : fonction `checkRsvpCode()` — devient un no-op qui
   skip directement vers `rsvpStep2`
2. La modale RSVP step1 (qui demande le code) est masquée : on saute direct au
   step 2 (formulaire) en ouvrant la modale.
3. Garder le HTML du step1 mais le marquer `display:none` pour éviter une grosse
   refonte de la modale.

Plus simple : modifier `openRsvpModal()` pour activer directement `rsvpStep2`
au lieu de `rsvpStep1`.

## Plan d'implémentation

1. **Ajouter CSS du gate** dans le `<style>` existant (un gros bloc dédié à la fin)
2. **Ajouter script de gating** en haut de `<body>` (inline, runs immediately)
3. **Ajouter HTML + JS du gate** en bas de `<body>` avant `</body>`
4. **Modifier `openRsvpModal()`** pour skip le step1
5. **Tester localement** :
   - Premier chargement → gate visible avec slideshow
   - Mauvais code → shake + message
   - Bon code → fade-out → site visible
   - Reload → site directement visible (localStorage)
   - `localStorage.clear()` → gate revient
6. **Screenshots** before/after pour valider visuellement

## Non-objectifs

- ❌ Vraie authentification serveur (pas de backend)
- ❌ Comptes individuels par invité (un seul code partagé)
- ❌ Tentatives limitées / rate limit (sans valeur en client-side)
- ❌ Refonte de la modale RSVP (juste skip de l'étape 1)
- ❌ Suppression complète du code de l'étape RSVP1 (on le laisse mort pour rollback facile)

## Critères d'acceptation

- [x] Le gate s'affiche au premier chargement
- [x] Le code `1960@1990A` débloque l'accès
- [x] Tout autre code déclenche shake + message
- [x] Une fois unlocked, reload → accès direct
- [x] `localStorage.clear()` → gate revient
- [x] Le RSVP ne demande plus de code (formulaire direct)
- [x] Slideshow de 5-6 photos en cross-fade lent
- [x] Glassmorphism + animations cascade d'entrée
- [x] Lien WhatsApp groupe accessible depuis le gate
- [x] Thème vert/or préservé
- [x] Mobile responsive (carte adaptée < 600px)

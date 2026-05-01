# Workflow: FantaBeach Base — Carte, Shop e Collezione
# Invocazione: /07-fanta-beach-base

> 🔜 **COMING SOON — Da implementare in v2+.**
> Questo workflow documenta la visione futura del FantaBeach per riferimento. Nella v1 la sezione FantaBeach mostra un placeholder "Coming Soon" nella navigazione. Non eseguire questo workflow finché non viene rimosso questo avviso da `@project-context`.

Leggi le skill `@fanta-beach`, `@auth-ruoli`, `@react-patterns`, `@typescript-expert` e `@api-design-principles` prima di procedere.

Il tuo obiettivo è implementare la parte base del FantaBeach: il catalogo carte, lo shop per acquistarle con i punti e la collezione personale dell'utente.

> ⚠️ **FantaBeach è un gioco virtuale, completamente separato dai Tornei fisici.** Non collegare le feature FantaBeach a eventi fisici reali o alla sezione Tornei. Gli eventi FantaBeach sono creati manualmente dagli admin con logica indipendente.

## Cosa devi produrre

### 1. Placeholder v1
Prima di implementare qualsiasi feature, crea `src/pages/FantaBeach/index.tsx` come semplice pagina placeholder con messaggio "Coming Soon 🔜" e breve descrizione del gioco. Questa pagina è quella attiva nella v1.

### 2. Edge Function purchase-card
Crea `/supabase/functions/purchase-card/index.ts`. Riceve `card_id`. Verifica che: l'utente sia autenticato, la carta esista ed `is_available: true`, il `points_balance` dell'utente sia sufficiente. In una transazione atomica: scala `points_balance` in `profiles` del costo della carta, crea il record in `user_cards`. **NON scalare `total_points_earned`** — i punti spesi non vengono sottratti dalla classifica. Crea un record in `point_transactions` con `amount` negativo e `reason: 'card_purchase'`. Leggi `@fanta-beach` per lo schema esatto e `@auth-ruoli` per i controlli di permesso.

### 3. Hook useCards
Crea `src/hooks/useCards.ts`. Carica il catalogo completo delle carte disponibili (`is_available: true`) ordinate per rarità e costo. Espone: `cards`, `loading`. Leggi `@api-design-principles` per la query ottimizzata.

### 4. Hook useCollection
Crea `src/hooks/useCollection.ts`. Carica tutte le carte possedute dall'utente loggato (`user_cards` join `cards`). Espone: `collection`, `loading`, `hasCard(cardId)`, `cardCount(cardId)` (quante copie possiede). Leggi `@fanta-beach` per lo schema e `@auth-ruoli` per la RLS su `user_cards`.

### 5. Hook useBalance
Crea `src/hooks/useBalance.ts`. Espone `pointsBalance` e `totalPointsEarned` del profilo utente in tempo reale. Usa Realtime per aggiornarsi automaticamente quando i punti cambiano (dopo una vittoria o un acquisto). Scrivi un commento in italiano che spiega la differenza tra i due valori: `points_balance` è il saldo spendibile nello shop (scende con gli acquisti), `total_points_earned` è il punteggio cumulativo per la classifica globale (non scende mai).

### 6. Componente CardItem
Crea `src/features/fanta-beach/CardItem/index.tsx`. Rappresenta una singola carta come una figurina: immagine del giocatore (da Supabase Storage bucket `carte-giocatori`), nome giocatore, badge rarità con colore (comune = grigio, raro = blu, leggendario = oro). Accetta una prop `variant`: `shop` (mostra il costo in punti e bottone acquista) o `collection` (mostra quante copie possiede l'utente). Gestisci il caso in cui la carta non ha ancora un'immagine con un placeholder elegante (le grafiche sono ancora in produzione). Leggi `@react-patterns` per i pattern di card component con varianti.

### 7. Componente ShopPage
Crea `src/features/fanta-beach/ShopPage/index.tsx`. Mostra il saldo punti spendibili (`points_balance`) in evidenza in cima. Griglia di `CardItem` in variante `shop`. Filtri per rarità. Quando l'utente clicca "Acquista" su una carta: mostra dialog di conferma con nome carta e costo, poi chiama la Edge Function `purchase-card`. Aggiorna il saldo automaticamente dopo l'acquisto. Leggi `@react-patterns` per griglie filtrabili.

### 8. Componente CollectionPage
Crea `src/features/fanta-beach/CollectionPage/index.tsx`. Griglia di tutte le carte possedute dall'utente. Le carte non possedute sono mostrate in grigio/silhouette (per dare l'effetto album figurine incompleto). Badge con il numero di copie se l'utente ne possiede più di una. Filtri per rarità e "possedute / non possedute". Leggi `@react-patterns` per griglie con stati vuoti e silhouette.

### 9. Pagina FantaBeach (v2)
Aggiorna `src/pages/FantaBeach/index.tsx` sostituendo il placeholder. Tab navigation interna con 3 tab: "Shop", "Collezione", "Rosa" (la Rosa sarà implementata nel workflow `/08-fanta-beach-avanzato`). Il tab Rosa mostra per ora un placeholder "Prossimamente". Mostra sempre in cima il saldo punti e i punti totali guadagnati.

## Regole da rispettare

- L'acquisto carte avviene **solo** via Edge Function — mai un insert diretto su `user_cards` dalla UI (vedi `@fanta-beach`)
- `total_points_earned` non deve mai essere scalato dagli acquisti — solo `points_balance` (vedi `@project-context`)
- Le immagini delle carte vengono dal bucket `carte-giocatori` di Supabase Storage — usa URL pubblici
- Gestisci sempre il caso in cui la carta non ha ancora un'immagine con un placeholder elegante
- Mobile-first: la griglia carte deve essere 2 colonne su mobile, 3-4 su desktop
- **Non collegare mai le feature FantaBeach ai Tornei fisici** — sono sistemi completamente separati

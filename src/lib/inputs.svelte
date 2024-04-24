<script lang="ts">
    import Trades from "./trades.svelte";

    let feeRate: number = 0.1;
    let pair: string = 'BTC/USD';
    let buyPrice: number = 65000;
    let stake: number = 100;
    let threshold:  number = 100;
    let feeRates = [0.1, 0.12, 0.14, 0.2, 0.22, 0.24, 0.25, 0.35, 0.4];
    let pairs = ['BTC/USD', 'BTC/EUR', 'ETH/EUR', 'ETC/EUR', 'BTC/USDT'];
    $: splitPair = pair.split('/');
    $: crypto = splitPair[0];
    $: currency = splitPair[1];

</script>



<label for="pairs">Paire = </label>
<select id="pairs" name="pairs" bind:value={pair}>
    {#each pairs as p }
        <option value={p}>&nbsp;{p}</option>
    {/each}
</select>
<br/>
<label for="feeRates">Frais = </label>
<select id="feeRates" name="feeRates"  bind:value={feeRate}>
    {#each feeRates as f }
        <option value={f}>{f} %</option>
    {/each}
</select>
<br/>
<br/>
<label for="price">Prix d'achat {crypto} en {currency} = </label>
<input type="number" id="price" name="price" bind:value={buyPrice} min="0"> 
<br/>
<label for="stake">Mise en {currency} = </label>
<input type="number" id="stake" name="stake" bind:value={stake} min="0"> 
<br/>
<label for="threshold">Seuil de gain en {currency} = </label>
<input type="number" id="threshold" name="threshold" bind:value={threshold} min="0"> 
<br/>
<div id="data">
    <Trades {feeRate} {pair} {buyPrice} {stake} {pairs} {currency} {crypto} {threshold} />
</div>
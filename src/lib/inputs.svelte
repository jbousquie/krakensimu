<script lang="ts">
    import Trades from "./trades.svelte";

    let feeRate: number = 0.1;
    let pair: string = 'BTC/USD';
    let buyPrice: number = 65000;
    let stake: number = 100;
    let threshold:  number = 100;
    let feeRates = [0.1, 0.12, 0.14, 0.2, 0.22, 0.24, 0.25, 0.35, 0.4];
    let pairs = ['BTC/USD', 'BTC/EUR', 'ETH/EUR', 'ETC/EUR', 'BTC/USDT'];
    let ratioX: number = 3;
    let ratioY: number = 0.3;
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
<input type="number" id="threshold" name="threshold" bind:value={threshold} min="0"> <span id="signal" class="pos"></span>
<br/>
<div id="graphContainer">
    
    <input type="range" id="ratioX" name="ratioX" bind:value={ratioX} min="1" max="10" step="0.5"><label for="ratioX">RatioX = {ratioX}</label><br/>
    <input type="range" id="ratioY" name="ratioY" bind:value={ratioY} min="0.01" max="1.0" step="0.01"><label for="ratioY">RatioY = {ratioY}</label><br/> 

    <svg id="graph" width="800" height="400" style="background-color:aliceblue;" viewBox="0 0 400 200" xmlns="http://www.w3.org/2000/svg">
    </svg>
</div>
<div id="data">
    <Trades {feeRate} {pair} {buyPrice} {stake} {pairs} {currency} {crypto} {threshold} {ratioX} {ratioY}/>
</div>
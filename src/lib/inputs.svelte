<script lang="ts">
    import Trades from "./trades.svelte";

    let feeRate: number = 0.1;
    let pair: string = 'BTC/USD';
    let buyPrice: number = 65000;
    let stake: number = 100;
    let feeRates = [0.1, 0.12, 0.14, 0.2, 0.22, 0.24, 0.25, 0.35, 0.4];
    let pairs = ['BTC/USD', 'BTC/EUR', 'ETH/EUR'];

</script>

<label for="feeRates">Frais </label>
<select id="feeRates" name="feeRates"  bind:value={feeRate}>
    {#each feeRates as f }
        <option value={f}>{f} %</option>
    {/each}
</select>

<label for="pairs">Paire </label>
<select id="pairs" name="pairs" bind:value={pair}>
    {#each pairs as p }
        <option value={p}>{p}</option>
    {/each}
</select>
<br/>
<br/>
<label for="price">Prix d'achat </label>
<input type="number" id="price" name="price" bind:value={buyPrice} min="0">

<label for="stake">Mise </label>
<input type="number" id="stake" name="stake" bind:value={stake} min="0">
<br/>
<div id="data">
    <Trades {feeRate} {pair} {buyPrice} {stake} {pairs} />
</div>
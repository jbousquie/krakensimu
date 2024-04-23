<script lang="ts">
    // https://docs.kraken.com/websockets-v2/#trade
    import Trades from "$lib/trades.svelte";
    import { onMount } from "svelte";

    export let feeRate: number = 0.1;
    export let pair: string = 'BTC/USD';
    export let buyPrice: number = 1000;
    export let stake: number = 100;
    export let pairs: [string];
    
    $: currency = pair.split('/')[1];
    $: entryFee = feeRate * stake * 0.01;
    $: sellPrice = buyPrice;
    $: variation = (sellPrice - buyPrice) / buyPrice;
    $: grossValue = (variation + 1) * (stake - entryFee);
    $: exitFee = feeRate * grossValue * 0.01;
    $: netValue = grossValue - stake - exitFee;

    let connecte = false;
    let trades: [{symbol: string; side: string; price: number; qty: number; ord_type: string}];
    let tradeList = '';

    onMount(() => {
        var socket: WebSocket = new WebSocket("wss://ws.kraken.com/v2");
        var connection_id: number = 0;
        var subscribe = {
            method: "subscribe",
            params: {
                channel: "trade",
                snapshot: true,
                symbol: pairs
            },
            req_id : 0
        };
        var unsubscribe = {
            method: "unsubscribe",
            params: {
                channel: "trade",
                symbol: pairs
            },
            req_id: 0,
        };

        socket.onmessage = function(e) {
            var msg = JSON.parse(e.data);

            // si le message a un channel
            if (msg.channel) {
                switch (msg.channel) {
                    // message de connexion initial contenant le req_id qu'on récupère
                    case 'status': 
                        if (msg.data && msg.data[0] && msg.data[0].connection_id) {
                            connection_id = parseInt(msg.data[0].connection_id);
                            subscribe.req_id = connection_id;
                            unsubscribe.req_id = connection_id;
                            connecte = true;
                        }
                        // souscription initiale
                        var payloadJson = JSON.stringify(subscribe);
                        socket.send(payloadJson);
                        break;
                    
                    // message de trade
                    case 'trade':
                        trades = msg.data;
                        let tradePair = trades[0].symbol;
                        if (tradePair == pair) {
                            sellPrice = trades[trades.length - 1].price;
                            tradeList = '';
                            for (let i = 0; i < trades.length; i++) {
                                let trd = trades[i];
                                let line = trd.symbol + " ordre = " + trd.ord_type + " prix = " + trd.price + " " + currency + " qty = " + trd.qty + " type ordre = " + trd.ord_type;
                                tradeList = tradeList + "<p>" + line + "</p>";
                            }
                        }
                        break;
                }
            }
        }
        
        socket.onclose = function(e) {
            var payloadJson = JSON.stringify(unsubscribe);
            socket.send(payloadJson);
        }

    });
</script>
<div id="status">
    {#if connecte } 
        <span>Connecté à Kraken</span>
    {/if}
</div>
<br/>
<div>
    frais à l'achat : {entryFee.toFixed(2)} {currency}<br/>
    prix de la cryto : {sellPrice.toFixed(2)} {currency}<br/>
    variation du prix depuis l'achat : {variation.toFixed(3)} %<br/>
    valeur actuelle de la mise : <b>{grossValue.toFixed(2)} {currency}</b><br/>
    frais de revente : {exitFee.toFixed(2)} {currency} <br/>
    <b>gain net : {netValue.toFixed(2)} {currency}</b><br/>
</div>
<br/>
<div>
    <u>TRADES LIVE</u>
        {@html tradeList}
</div>

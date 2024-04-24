

<script lang="ts">
    // https://docs.kraken.com/websockets-v2/#trade
    import { onMount } from "svelte";

    export let feeRate: number = 0.1;
    export let pair: string;
    export let buyPrice: number = 1000;
    export let stake: number = 100;
    export let pairs: any;
    export let crypto: string;
    export let currency: string;
    export let threshold: number = 100;
    

    $: entryFee = feeRate * stake * 0.01;
    $: sellPrice = 100000;
    $: variation = (sellPrice - buyPrice) / buyPrice;
    $: grossValue = (variation + 1) * (stake - entryFee);
    $: exitFee = feeRate * grossValue * 0.01;
    $: netGain = grossValue - stake - exitFee;
    $: plusVariation = (variation >= 0) ? '+' : '';
    $: plusGain = (netGain >= 0) ? '+' : '';
    $: sign = (netGain >= 0) ? 'pos' : 'neg';
    $: checkGainThreshold();

    let maxTradeNb: number = 25;
    let connecte: boolean = false;
    let title: string = 'SimuKraken';
    let trades: [{symbol: string; side: string; price: number; qty: number; ord_type: string}];
    let tradeList: string = '';

    // derniers prix : on tient à jour un tableau de 100 entrées par paire
    let lastPrices: any = {};
    for (let i = 0; i < pairs.length; i++) {
        let pr = pairs[i];
        lastPrices[pr] = [];
    }

    // test si gain >= seuil
    function checkGainThreshold() {
        if (netGain >= threshold) {
            notifyThreshold();
        } else {
            notifyReset();
        }
    }
    // notifie à chaque fois que le gain passe au dessus du seuil
    function notifyThreshold() {
        let msgGain = "Gain : " + netGain.toFixed(2) + " " + currency + " !!!"
        document.title = (document.title == title) ? msgGain : title;       
    }

    // supprime la notification de seuil dépassé
    function notifyReset() {
        document.title = title;
    }


    // retourne une string html des trades dans l'ordre chrono (inversé)
    function listTrades(lastPricePair: any): string {
        let listTrade = '<br/>';
        for (let i = lastPricePair.length - 1; i >= 0; i--) {
            let trd = lastPricePair[i];
            let line = trd.line;                                
            let cls = (trd.side == "sell") ? "sell" : 'buy';
            listTrade = listTrade + "<span class=" + cls +">" + line + "</span><br/>";
        }
        return listTrade;
    }

    // fonction qui renvoie le  caractère ⮝ ou ⮟  ou "&nbsp;" selon le delta curent - last
    function getMove(last: number, current: number): string {
        let move = "&nbsp;";
        if (current - last > 0) {
            move = "⮝";
        } else if (last - current > 0) {
            move = "⮟";
        }
        return move;
    }

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
                        // on stoke les nouvelles lignes de trade dans le tableau de la paire
                        for (let i = 0; i < trades.length; i++) {
                            let trd = trades[i];
                            let prev;
                            if (i > 0) {                               
                                prev = trades[i  - 1];          // on compare les prix au sein des trades reçus
                            } else {
                                let l = lastPrices.length;
                                if (l > 0) {
                                    prev = lastPrices[l - 1]; // on compare le premier trade reçu à celui reçu précédemment
                                } else {
                                    prev = {price: trd.price};
                                }
                            }
                            let move = getMove(prev.price, trd.price);

                            let sd = (trd.side == "buy") ? "buy " : trd.side;
                            let tp = (trd.ord_type == "limit") ? " limit" : trd.ord_type;

                            let line: string = trd.symbol + " " + sd + "/" + tp + "&nbsp;&nbsp;&nbsp;<b>" + trd.price.toFixed(2) + " " + move +"</b>&nbsp;&nbsp;&nbsp;" + trd.qty.toFixed(4) ;
                            let elem = { price: trd.price,  side: trd.side, line: line };
                            lastPrices[tradePair].push(elem);
                            if (lastPrices[tradePair].length >= maxTradeNb) {
                                lastPrices[tradePair].shift();
                            }
                        }

                        // on affiche les données de la paire sélectionnée
                        if (tradePair == pair) {
                            let lastPricePair = lastPrices[pair];
                            let lastIndex = lastPricePair.length - 1;
                            sellPrice = lastPricePair[lastIndex].price;
                            tradeList = listTrades(lastPricePair);
                        }
                        break;
                    
                    // heartbeat
                    case 'heartbeat':
                        let lastPricePair = lastPrices[pair];
                        let lastIndex = lastPricePair.length - 1;
                        sellPrice = lastPricePair[lastIndex].price;
                        tradeList = listTrades(lastPricePair);
                        checkGainThreshold();
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
{#if connecte}
<br/>
 <div class="buy">Connecté à Kraken</div>
{/if}
<br/>
<div>
    frais à l'achat : {entryFee.toFixed(2)} {currency}<br/>
    prix de la crypto {crypto} : <b>{sellPrice.toFixed(2)} {currency}</b><br/>
    variation du prix par rapport à l'achat : {plusVariation}{variation.toFixed(3)} %<br/>
    valeur actuelle de la mise : <b>{grossValue.toFixed(2)} {currency}</b><br/>
    frais de revente : {exitFee.toFixed(2)} {currency} <br/>
    <b>gain net : <span class={sign}>{plusGain}{netGain.toFixed(2)} {currency}</span></b><br/>
</div>
<br/>
<div>
    <u>TRADES LIVE</u>
    <div class="list">
        {@html tradeList}
    </div>
</div>

<style>
    .pos {
        color: green;
    }
    .neg {
        color: red;
    }
    .list {
        font-family: 'Courier New', Courier, monospace;
        font-size: 15px;
    }
</style>

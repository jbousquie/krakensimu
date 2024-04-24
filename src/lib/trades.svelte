

<script lang="ts">
    // https://docs.kraken.com/websockets-v2/#trade
    import { onMount } from "svelte";

    // Une ligne de trade stockée dans un tableau pour affichage et calculs
    class TradeLine {
        price: number = 0;
        side: string = ''; 
        line: string = ''; 
        timestamp: string ='';
        constructor(price: number, side: string, timestamp: string, line: string) {
            this.price = price;
            this.side = side;
            this.timestamp = timestamp;
            this.line = line;
        }
    }

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

    let maxTradeNb: number = 250;
    let connecte: boolean = false;
    let title: string = 'SimuKraken';
    let signalChar = '⬤';
    let trades: [{symbol: string; side: string; price: number; qty: number; ord_type: string; timestamp: string}]; // Réponse kraken
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
        let signal = document.querySelector("#signal");
        if (signal) {
            signal.innerHTML = (signal.innerHTML == signalChar) ? "" : signalChar;
        }
    }

    // supprime la notification de seuil dépassé
    function notifyReset() {
        document.title = title;
        let signal = document.querySelector("#signal");
        if (signal) {
            signal.innerHTML = "";
        }
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
                                prev = trades[i  - 1];                  // on compare les prix au sein des trades reçus
                            } else {
                                let l = lastPrices.length;
                                if (l > 0) {
                                    prev = lastPrices[tradePair][l - 1]; // on compare le premier trade reçu à celui reçu précédemment
                                } else {
                                    prev = {price: trd.price};           // mock objet s'il n'y avait pas de prev trade
                                }
                            }
                            let move = getMove(prev.price, trd.price);  // on en déduit si le prix monte ou descend

                            let sd = (trd.side == "buy") ? "buy " : trd.side;
                            let tp = (trd.ord_type == "limit") ? " limit" : trd.ord_type;

                            let line: string = trd.symbol + " " + sd + "/" + tp + "&nbsp;&nbsp;&nbsp;<b>" + trd.price.toFixed(2) + " " + move +"</b>&nbsp;&nbsp;&nbsp;" + trd.qty.toFixed(4) ;
                            let tradeLine = new TradeLine(trd.price, trd.side, trd.timestamp, line);
                            lastPrices[tradePair].push(tradeLine);
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
                        displayGraph(tradePair);
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


    function displayGraph(pair: string) {
        let lastPricePair: [TradeLine] = lastPrices[pair];
        let coords = [];
        let lastx: number = 0;
        let lasty: number = 0;
        let maxx: number = 0;
        let maxy: number = 0;
        let minx: number = Number.MAX_SAFE_INTEGER;
        let miny: number = Number.MAX_SAFE_INTEGER;
        for (let i = lastPricePair.length - 1; i >= 0; i--) {    // ordre chrono = inverse
            let trd: TradeLine = lastPricePair[i];
            let x: number = +Date.parse(trd.timestamp);
            let y: number = trd.price;
            if (x == lastx && y == lasty) {
                continue;                   // on ne stocke pas les doublons
            }
            coords.push(x * 0.0000000001, y * 0.001);
            lastx = x;
            lasty = y;
            if (x > maxx) { maxx = x; }
            if (y > maxy) { maxy = y; }
            if (x < minx) { minx = x; }
            if (y < miny) { miny = y; }
        }
        
        let d = "M " + coords[0].toString() + " " + coords[1].toString();
        for (let i = 2; i < coords.length; i = i + 2) {
            d = d + " L " + coords[i].toString() + " " + coords[i + 1].toString();
        }
        let path = '<path d="' + d + '" stroke-width="2" stroke="black" />';
        //let path ='<circle cx="' + minx * 0.00000000001 + '" cy="'+ miny / 10000 + '" r="50" stroke="red" stroke-width="10"/>'; 
        // https://developer.mozilla.org/fr/docs/Web/SVG/Tutorial/Positions
        let viewbox = Math.floor(minx).toString() + " " + Math.floor(miny).toString() + " " + Math.floor(maxx).toString() + " " + Math.floor(maxy).toString();
        let svg = document.querySelector("#graph");
        if (svg) {
            viewbox = "180 50 20 20";
            svg.innerHTML = path;
            svg.setAttribute("viewBox", viewbox);
        }
    }
</script>
{#if connecte}
<br/>
 <div class="pos">Connecté à Kraken</div>
{/if}
<br/>
<div>
    prix actuel de la crypto {crypto} : <b>{sellPrice.toFixed(2)} {currency}</b><br/>
    valeur actuelle de la mise : <b>{grossValue.toFixed(2)} {currency}</b><br/><br/>
    <b><span class={sign}>gain net : {plusGain}{netGain.toFixed(2)} {currency}</span></b><br/><br/>
    variation du prix de {crypto} par rapport à l'achat : {plusVariation}{(100 * variation).toFixed(2)} %<br/>
    frais à l'achat : {entryFee.toFixed(2)} {currency} (déjà décomptés)<br/>
    frais de revente : {exitFee.toFixed(2)} {currency} (si revente à ce prix)<br/><br/>
</div>
<br/>
<div>
    <u>TRADES LIVE</u>
    <div class="list">
        {@html tradeList}
    </div>
</div>

<svg id="graph" height="400" width="1200" style="background-color:aliceblue;" xmlns="http://www.w3.org/2000/svg">
</svg>

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



<script lang="ts">
    // https://docs.kraken.com/websockets-v2/#trade
    import { onMount } from "svelte";

    // Une ligne de trade stockée dans un tableau historique
    class TradeLine {
        price: number = 0;
        qty: number = 0;
        side: string = ''; 
        typord: string = '';
        move: string = ''; 
        timestamp: string ='';
        constructor(price: number, qty: number, side: string, typord: string, timestamp: string, move: string) {
            this.price = price;
            this.qty = qty;
            this.side = side;
            this.typord = typord;
            this.timestamp = timestamp;
            this.move = move;
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
    export let ratioX: number = 5;
    export let ratioY: number = 0.5;
    

    $: entryFee = feeRate * stake * 0.01;
    $: sellPrice = 100000;
    $: variation = (sellPrice - buyPrice) / buyPrice;
    $: grossValue = (variation + 1) * (stake - entryFee);
    $: exitFee = feeRate * grossValue * 0.01;
    $: netGain = grossValue - stake - exitFee;
    $: plusVariation = (variation >= 0) ? '+' : '';
    $: plusGain = (netGain >= 0) ? '+' : '';
    $: sign = (netGain >= 0) ? 'pos' : 'neg';
    $: resetGrid = (pair) ? true : true; // si la paire change alors il faut rescaler le graphique

    let maxTradeNb: number = 3000;       // nb de trades max dans l'historique
    let maxDisplayNb: number = 30;      // nd de trades max à afficher
    let connecte: boolean = false;
    let title: string = 'SimuKraken';
    let signalChar = '⬤';
    let trades: [{symbol: string; side: string; price: number; qty: number; ord_type: string; timestamp: string}]; // Réponse kraken
    let tradeList: string = '';  // string html d'affichage des trades
    let maxX: number = 0;
    let maxY: number = 0;
    let minX: number = Number.MAX_VALUE;
    let minY: number = Number.MAX_VALUE;
    

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
    function listTrades(pair: string, lastPricePair: any, maxNb: number): string {
        let listTrade = '<br/>';
        let limit = 0;
        let len = lastPricePair.length - 1;
        if (len - maxNb > 0) {
            limit = len - maxNb + 1;
        }
        for (let i = len; i >= limit; i--) {
            let trd = lastPricePair[i];
            let sd = (trd.side == "buy") ? "buy " : trd.side;
            let tp = (trd.typord == "limit") ? " limit" : trd.typord;
            let cls = (trd.side == "sell") ? "sell" : 'buy';
            let line: string = pair + " " + sd + "/" + tp + "&nbsp;&nbsp;&nbsp;<b>" + trd.price.toFixed(2) + " " + trd.move +"</b>&nbsp;&nbsp;&nbsp;" + trd.qty.toFixed(4) ;            
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
                                let l = lastPrices[tradePair].length;
                                if (l > 0) {
                                    prev = lastPrices[tradePair][l - 1]; // on compare le premier trade reçu à celui reçu précédemment
                                } else {
                                    prev = {price: trd.price};           // mock objet s'il n'y avait pas de prev trade
                                }
                            }
                            let move = getMove(prev.price, trd.price);  // on en déduit si le prix monte ou descend
                            let tradeLine = new TradeLine(trd.price, trd.qty, trd.side, trd.ord_type, trd.timestamp, move);
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
                            tradeList = listTrades(pair, lastPricePair, maxDisplayNb);
                        }
                        
                        break;
                    
                    // heartbeat
                    case 'heartbeat':
                        let lastPricePair = lastPrices[pair];
                        let lastIndex = lastPricePair.length - 1;
                        sellPrice = lastPricePair[lastIndex].price;
                        //tradeList = listTrades(lastPricePair);
                        checkGainThreshold();
                        displayGraph(pair, ratioX, ratioY);
                        break;
                }
            }
        }
        
        socket.onclose = function(e) {
            var payloadJson = JSON.stringify(unsubscribe);
            socket.send(payloadJson);
        }

    });


    function displayGraph(pair: string, ratioX: number, ratioY: number) {
        let lastPricePair: [TradeLine] = lastPrices[pair];
        let coords: any = [];                                   // tableau de coordonnées pour le graphe
        let lastx: number = 0;
        let lasty: number = 0;    let maxx: number = 0;
        let maxy: number = 0;
        let minx: number = Number.MAX_VALUE;
        let miny: number = Number.MAX_VALUE;

        // stockage des coordonnées brutes des trades
        for (let i = 0; i < lastPricePair.length; i++) {       
            let trd: TradeLine = lastPricePair[i];
            let x: number = +Date.parse(trd.timestamp);
            let y: number = trd.price;
            if (x == lastx && y == lasty) {
                continue;                   // on ne stocke pas les doublons
            }
            lastx = x;
            lasty = y;
            if ( x > maxx) { maxx = x; }
            if ( y > maxy) { maxy = y; }
            if (x < minx)  { minx = x; }
            if (y < miny)  { miny = y; }
            coords.push(x);
            coords.push(y);
        }
        if (resetGrid) {
            maxX = maxx;
            maxY = maxy;
            minX = minx;
            minY = miny;
        }
        resetGrid = false;
        let stepNb = 10;
        let stepSize = 200 / stepNb;
        let mid = (maxY - minY) * 0.5 + minY;
        let midshift = mid % stepSize;
        // translation/scale des coordonnées
        for (let i = 0; i < coords.length; i = i + 2) {
            coords[i] = (coords[i] - minX) * 0.0001 * ratioX;
            coords[i + 1] = 100 - (coords[i + 1] - mid + midshift) * ratioY;
        }
        // path de la courbe
        let d = "M " + coords[0].toString() + " " + coords[1].toString();
        for (let i = 2; i < coords.length; i = i + 2) {
            d = d + " L " + coords[i].toString() + " " + coords[i + 1].toString();
        }
        // grille
        let svgCommands = '';
        let value = Math.floor(mid / stepSize) * stepSize + (stepSize * stepNb / 2); // on commence à stepNb/2 intervalles au dessus de la valeur mid arrondie
        for (let s = 0; s < stepNb; s++) {
            let l = (s + 1)  * stepSize;
            let hgrid = "<path d=\"M 0 " + l.toString() + " H 400\" fill=\"transparent\" stroke=\"lightgrey\" stroke-width=\"0.3\" />";
            svgCommands = svgCommands + hgrid;
            value -= stepSize * ratioY;
            let hgridprice = '<text x="' + 5 + '" y="' + (l - 2).toString() + '" fill="lightgrey">' + value.toFixed(2) + '</text>';
            svgCommands =svgCommands + hgridprice;
        }
        svgCommands = svgCommands + '<path d="' + d + '" stroke-width="1" fill="transparent" stroke="blue" />';
        let lastPrice = lastPricePair[lastPricePair.length - 1].price;
        let shift = 3;
        let ln = coords.length - 2;
        let x = (coords[ln] + shift).toString();
        let y = (coords[ln + 1] - shift).toString();
        svgCommands = svgCommands + '<text x="' + x + '" y="' + y + '" fill="blue">' + lastPrice.toString() + '</text>';
        // https://developer.mozilla.org/fr/docs/Web/SVG/Tutorial/Positions
        let svg = document.querySelector("#graph");
        if (svg) {
            svg.innerHTML = svgCommands;
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

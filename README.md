# pastebi

user: pass: https://logsearch.sys.payhub.hcloud.santanderuk.pre.corp
T0yF0CHN1Ky2H8_zE9AyR3qdpS9WTU7s

/*
 * Copyright 2018 Goldman Sachs.
 */
package com.gs.futures.refdata.services.prime.price;

import java.math.BigDecimal;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CountDownLatch;

public class ModernPriceHolder {

    private Map<String, PriceThroughTime> vault = new ConcurrentHashMap<>();

    /** Called when a price ‘p’ is received for an entity ‘e’ */
    public void putPrice(String e, BigDecimal p) {

        vault.compute(e, (s, priceThroughTime) -> {
            if(priceThroughTime == null) {
                priceThroughTime = new PriceThroughTime();
            }
            priceThroughTime.setPricePut(p);
            priceThroughTime.notifyRequesters();
            return priceThroughTime;
        });
    }


    /** Called to get the latest price for entity ‘e’ */
    public BigDecimal getPrice(String e) {
        return vault.computeIfAbsent(e, s -> new PriceThroughTime()).updatePriceGot();
    }

    /**
     * Called to determine if the price for entity ‘e’ has changed since the
     * last call to getPrice(e).
     */
    public boolean hasPriceChanged(String e) {
        return vault.computeIfAbsent(e, s -> new PriceThroughTime()).hasPriceChanged();
    }

    /**
     * Returns the next price for entity ‘e’. If the price has changed since the
     * last call to getPrice() or waitForNextPrice(), it returns immediately
     * that price. Otherwise it blocks until the next price change for entity
     * ‘e’.
     */
    public BigDecimal waitForNextPrice(String e) throws InterruptedException {

        CountDownLatch countDownLatch = new CountDownLatch(1);

        PriceThroughTime priceThroughTimeComputed = vault.compute(e, (s, priceTt) -> {
            if(priceTt == null) {
                PriceThroughTime priceThroughTime = new PriceThroughTime();
                priceThroughTime.addAnotherRequester(countDownLatch);
                return priceThroughTime;
            }
            if(priceTt.hasPriceChanged()) {
                countDownLatch.countDown(); // no need to wait
            } else {
                priceTt.addAnotherRequester(countDownLatch);
            }
            return priceTt;
        });

        countDownLatch.await();
        return priceThroughTimeComputed.updatePriceGot();
    }

}

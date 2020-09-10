
## **new field to save current push status**
Push_Status: PENDING, PUSHING, SUCCESS, FAIL

```SQL
ALTER TABLE ls_um_payment_ac_invoice_t ADD COLUMN push_status VARCHAR(20) DEFAULT 'PENDING' COMMENT 'push invoice to AC status';
```

![PUSH_STATUS_LIFE_CYCLE](./push_status.png)



## **backend process of pushing invoice**

1. **/invoice/push will just save an invoice list waiting to push**

    due to it's a simple process, api will response very quickly.

![invoice_push_api](./push_invoice_api.png)

2. **set an always-running-thread to push invoice async**
    

    ![process_thread](./process_thread.png)
    
3. **the way to confirm process thread alive**

    a. thread will log its status periodically

    b. monitor the number of pushing data

int zgt_tx::set_lock(long tid1, long sgno1, long obno1, int count, char lockmode1) {
    zgt_tx *tx = get_tx(tid1);  // Get transaction object

    zgt_p(0);  // Lock Transaction Manager to prevent race conditions
    zgt_hlink *pconnect = ZGT_Ht->find(sgno1, obno1);  // Check if object is locked

    // Case 1: If no other transaction holds a lock, grant it immediately
    if (pconnect == NULL) {
        ZGT_Ht->add(tx, sgno1, obno1, lockmode1); // Grant lock
        zgt_v(0);  // Unlock TM
        tx->perform_read_write_operation(tid1, obno1, lockmode1); // Execute operation
        return 0;
    }

    // Case 2: If the current transaction already holds the lock, continue execution
    if (tx->tid == pconnect->tid) {
        zgt_v(0);  // Unlock TM
        tx->perform_read_write_operation(tid1, obno1, lockmode1);
        return 0;
    }

    // Case 3: Shared Lock Compatibility Check
    if (pconnect->lockmode == 'S') {
        if (lockmode1 == 'S') {
            // Multiple readers are allowed → Proceed
            zgt_v(0);  // Unlock TM
            tx->perform_read_write_operation(tid1, obno1, 'S');
            return 0;
        } else {
            // If requesting Exclusive Lock (X), must wait
            tx->obno = obno1;
            tx->lockmode = 'X';
            tx->status = TR_WAIT;
            tx->setTx_semno(pconnect->tid, pconnect->tid);
            printf(":::Tx%d is waiting for exclusive access on Tx%d for object %d\n", tid1, pconnect->tid, obno1);
            fflush(stdout);
            zgt_v(0);  // Unlock TM before waiting
            zgt_p(pconnect->tid); // Block transaction until `pconnect->tid` releases the lock
        }
    }
    
    // Case 4: If the object is locked in Exclusive (`X`) mode, block the transaction
    if (pconnect->lockmode == 'X') {
        printf(":::Tx%d wants to access Object %d, but Tx%d holds an Exclusive Lock\n", tid1, obno1, pconnect->tid);
        fflush(stdout);

        tx->obno = obno1;
        tx->lockmode = lockmode1;
        tx->status = TR_WAIT;
        tx->setTx_semno(pconnect->tid, pconnect->tid);
        zgt_v(0); // Unlock TM before waiting

        zgt_p(pconnect->tid);  // Block T_new until T_existing releases the lock

        // Once lock is released, continue execution
        tx->obno = -1;
        tx->lockmode = ' ';
        tx->status = TR_ACTIVE;
        printf(":::Tx%d is resuming after waiting on Tx%d for Object %d\n", tid1, pconnect->tid, obno1);
        fflush(stdout);
        tx->perform_read_write_operation(tid1, obno1, lockmode1);
    }

    return 0;
}

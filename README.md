;; ZLK Jetton Wallet (TEP-74 compliant)
#include "stdlib.fc";

global slice owner_address;
global slice jetton_master_address;
global int balance;

;; ================= DATA =================

() load_data() impure {
    var ds = get_data().begin_parse();
    owner_address = ds~load_msg_addr();
    jetton_master_address = ds~load_msg_addr();
    balance = ds~load_coins();
}

() save_data() impure {
    var b = begin_cell();
    b.store_slice(owner_address);
    b.store_slice(jetton_master_address);
    b.store_coins(balance);
    set_data(b.end_cell());
}

;; =============== UTILS ==================

() send_msg(slice to, int value, int mode, cell body) impure inline {
    var msg = begin_cell()
        .store_uint(0x18, 6) ;; int msg
        .store_slice(to)
        .store_coins(value)
        .store_uint(0, 1)
        .store_ref(body)
        .end_cell();
    send_raw_message(msg, mode);
}

;; ============== TRANSFER ===============

() send_tokens(slice to, int amount, slice response, int fwd_ton, cell fwd_payload) impure {
    throw_unless(705, equal_slices(sender_address(), owner_address));
    throw_unless(706, balance >= amount);

    balance -= amount;

    var body = begin_cell()
        .store_uint(0xf8a7ea5, 32) ;; transfer
        .store_uint(0, 64) ;; query_id
        .store_coins(amount)
        .store_slice(owner_address)
        .store_slice(response)
        .store_coins(fwd_ton)
        .store_uint(1, 1)
        .store_ref(fwd_payload)
        .end_cell();

    send_msg(to, 0, 1, body);
    save_data();
}

;; ======== RECEIVE INTERNAL TRANSFER ========

() receive_tokens(slice from, int amount, slice response, int query_id) impure {
    throw_unless(707, equal_slices(sender_address(), jetton_master_address));

    balance += amount;

    if (!response.slice_empty?()) {
        var notify = begin_cell()
            .store_uint(0x7362d09c, 32) ;; transfer_notification
            .store_uint(query_id, 64)
            .store_coins(amount)
            .store_slice(from)
            .end_cell();

        send_msg(response, 0, 1, notify);
    }

    save_data();
}

;; ================= BURN =================

() burn(int amount, slice response, int query_id) impure {
    throw_unless(708, equal_slices(sender_address(), owner_address));
    throw_unless(709, balance >= amount);

    balance -= amount;

    var body = begin_cell()
        .store_uint(0x595f07bc, 32) ;; burn
        .store_uint(query_id, 64)
        .store_coins(amount)
        .store_slice(owner_address)
        .store_slice(response)
        .end_cell();

    send_msg(jetton_master_address, 0, 1, body);
    save_data();
}

;; ============= INTERNAL HANDLER =============

() recv_internal(int msg_value, cell in_msg, slice in_body) impure {
    load_data();

    if (in_body.slice_empty?()) {
        return ();
    }

    var op = in_body~load_uint(32);

    ;; internal_transfer
    if (op == 0x178d4519) {
        var query_id = in_body~load_uint(64);
        var amount = in_body~load_coins();
        var from = in_body~load_msg_addr();
        var response = in_body~load_msg_addr();
        receive_tokens(from, amount, response, query_id);
        return ();
    }

    ;; transfer
    if (op == 0xf8a7ea5) {
        var query_id = in_body~load_uint(64);
        var amount = in_body~load_coins();
        var to = in_body~load_msg_addr();
        var response = in_body~load_msg_addr();
        var fwd_ton = in_body~load_coins();
        var has_payload = in_body~load_uint(1);

        cell payload = null();
        if (has_payload == 1) {
            payload = in_body~load_ref();
        }

        send_tokens(to, amount, response, fwd_ton, payload);
        return ();
    }

    ;; burn
    if (op == 0x595f07bc) {
        var query_id = in_body~load_uint(64);
        var amount = in_body~load_coins();
        var response = in_body~load_msg_addr();
        burn(amount, response, query_id);
        return ();
    }
}

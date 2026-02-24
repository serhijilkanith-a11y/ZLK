;; ZLK Jetton Wallet (TEP-74)
#include "stdlib.fc";

global slice owner_address;
global slice jetton_master_address;
global int balance;

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

() send_tokens(slice to, int amount, slice response, int fwd_ton, cell fwd_payload) impure {
    throw_unless(705, equal_slices(sender_address(), owner_address));
    throw_unless(706, balance >= amount);

    balance -= amount;

    var msg_body = begin_cell()
        .store_uint(0xf8a7ea5, 32) ;; transfer
        .store_uint(0, 64)
        .store_coins(amount)
        .store_slice(owner_address)
        .store_slice(response)
        .store_coins(fwd_ton)
        .store_uint(1, 1)
        .store_ref(fwd_payload)
        .end_cell();

    send_raw_message(
        begin_cell()
            .store_uint(0x18, 6)
            .store_slice(to)
            .store_coins(0)
            .store_uint(4, 4)
            .store_ref(msg_body)
            .end_cell(),
        1
    );

    save_data();
}

() receive_tokens(slice from, int amount, slice response) impure {
    throw_unless(707, equal_slices(sender_address(), jetton_master_address));

    balance += amount;

    if (!response.slice_empty?()) {
        var msg = begin_cell()
            .store_uint(0x7362d09c, 32) ;; transfer_notification
            .store_uint(0, 64)
            .store_coins(amount)
            .store_slice(from)
            .end_cell();

        send_raw_message(
            begin_cell()
                .store_uint(0x10, 6)
                .store_slice(response)
                .store_coins(0)
                .store_uint(0, 1)
                .store_ref(msg)
                .end_cell(),
            1
        );
    }

    save_data();
}

() burn(int amount, slice response) impure {
    throw_unless(705, equal_slices(sender_address(), owner_address));
    throw_unless(706, balance >= amount);

    balance -= amount;

    var msg = begin_cell()
        .store_uint(0x7bdd97de, 32) ;; burn_notification
        .store_uint(0, 64)
        .store_coins(amount)
        .store_slice(owner_address)
        .store_slice(response)
        .end_cell();

    send_raw_message(
        begin_cell()
            .store_uint(0x18, 6)
            .store_slice(jetton_master_address)
            .store_coins(0)
            .store_uint(4, 4)
            .store_ref(msg)
            .end_cell(),
        1
    );

    save_data();
}

() recv_internal(int balance_ton, int msg_value, cell in_msg_full, slice in_msg_body) impure {
    load_data();

    if (in_msg_body.slice_empty?()) {
        return ();
    }

    var sender = in_msg_full.begin_parse()~load_msg_addr();
    var op = in_msg_body~load_uint(32);

    ;; transfer (від власника)
    if (op == 0xf8a7ea5) {
        var query_id = in_msg_body~load_uint(64);
        var amount = in_msg_body~load_coins();
        var to = in_msg_body~load_msg_addr();
        var response = in_msg_body~load_msg_addr();
        var fwd_ton = in_msg_body~load_coins();
        var has_payload = in_msg_body~load_uint(1);
        var payload = has_payload ? in_msg_body~load_ref() : null();

        send_tokens(to, amount, response, fwd_ton, payload);
        return ();
    }

    ;; internal transfer (від мінтера або іншого wallet)
    if (op == 0x178d4519) {
        var query_id = in_msg_body~load_uint(64);
        var amount = in_msg_body~load_coins();
        var from = in_msg_body~load_msg_addr();
        var response = in_msg_body~load_msg_addr();

        receive_tokens(from, amount, response);
        return ();
    }

    ;; burn
    if (op == 0x595f07bc) {
        var query_id = in_msg_body~load_uint(64);
        var amount = in_msg_body~load_coins();
        var response = in_msg_body~load_msg_addr();

        burn(amount, response);
        return ();
    }

    throw(0xffff);
}

(int, slice, slice, int) get_wallet_data() method_id {
    load_data();
    return (balance, owner_address, jetton_master_address, 0);
}

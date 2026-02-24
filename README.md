}
#ZLK
;; STON.fi LP Jetton Minter
#include "stdlib.fc";

global int total_supply;
global slice UQCO0tSfrvUCnAKGZQJ9v00V2R3RBzaaypXF9wlcRTfnJrG9
global cell jetton_content;
global cell jetton_wallet_code;

() load_data() impure {
    var ds = get_data().begin_parse();
    total_supply = ds~load_coins();
    admin_address = ds~load_msg_addr();
    jetton_content = ds~load_ref();
    jetton_wallet_code = ds~load_ref();
}

() save_data() impure {
    set_data(begin_cell()
        .store_coins(total_supply)
        .store_slice(admin_address)
        .store_ref(jetton_content)
        .store_ref(jetton_wallet_code)
        .end_cell());
}

slice calculate_user_wallet_address(slice owner_address) inline {
    return begin_cell()
        .store_uint(0, 2)
        .store_slice(owner_address)
        .store_uint(0, 1)
        .store_ref(jetton_wallet_code)
        .end_cell()
        .begin_parse();
}

() mint(slice to, int amount) impure {
    total_supply += amount;
    var wallet = calculate_user_wallet_address(to);

    send_raw_message(begin_cell()
        .store_uint(0x178d4519, 32)
        .store_slice(wallet)
        .store_coins(amount)
        .end_cell(), 64);
}

() burn(slice from, int amount) impure {
    total_supply -= amount;
}

() recv_internal(int msg_value, cell in_msg_full, slice in_msg_body) impure {
    load_data();

    if (in_msg_body.slice_empty?()) {
        return ();
    }

    var op = in_msg_body~load_uint(32);

    ;; mint by router
    if (op == 0x178d4519) {
        var sender = get_msg_sender();
        throw_unless(100, equal_slices(sender, admin_address));

        var to = in_msg_body~load_msg_addr();
        var amount = in_msg_body~load_coins();

        mint(to, amount);
        save_data();
        return ();
    }

    ;; burn notification
    if (op == 0x595f07bc) {
        var amount = in_msg_body~load_coins();
        burn(get_msg_sender(), amount);
        save_data();
        return ();
    }

    ;; get_jetton_data
    if (op == 0x2fcb26a2) {
        var b = begin_cell()
            .store_coins(total_supply)
            .store_slice(admin_address)
            .store_ref(jetton_content)
            .store_ref(jetton_wallet_code)
            .end_cell();
        send_raw_message(b, 64);
        return ();
    }

    throw(0xffff);
}

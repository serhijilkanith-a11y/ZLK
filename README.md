#ZLK
My crypto coin 
Supporting the TON eco-system
EQDnyVY_XdTtjg1cr6z_nf7f1gyL5niPlEyOPriHzSuX1Ctn
ZLK
9950000000
() recv_internal(int my_balance, int msg_value, cell in_msg_full, slice in_msg_body) impure {
    throw_if(error::empty_not_allowed, in_msg_body.slice_empty?());
    ctx::init(my_balance, msg_value, in_msg_full, in_msg_body);

    ;; 1bounced message
    if ctx.at(IS_BOUNCED) { 
        return ();
    }

    ;; workchain
    throw_unless(error::wrong_workchain, ctx.at(SENDER).address::check_workchain(params::workchain))
    storage::load();

    ;; token0 token 
    throw_if(error::invalid_token, equal_slices(storage::token0_address, storage::token1_address));

    let token_address = ctx.at(TOKEN);  
    if ! (equal_slices(token_address, storage::token0_address) || equal_slices(token_address, storage::token1_address))  
        log("Invalid token received, sending refund with reason")
        let msg = builder().store_uint(0xDEAD, 32) ;; 
                        .store_string("Invalid token")
                        .end_cell();

        ctx::send_message(ctx.at(SENDER), msg_value, msg);

        ;;  refund 
        ctx::send_refund(ctx.at(SENDER), msg_value);

        ;; throw
        return ();
    }

    ;; --- collect_fees ---
    if equal_slices(ctx.at(SENDER), storage::protocol_fee_address) {
        handle_protocolfee_messages();
        return ();
    }

    ;; --- swap, provide_lp та governance повідомлень ---
    if equal_slices(ctx.at(SENDER), storage::router_address) {
        handle_router_messages();
        return ();
    }

    ;; -- LP wallet messages ---
    if handle_lp_wallet_messages() {
        return ();
    }

    ;; --- LP account messages ---
    if handle_lp_account_messages() {
        return ();
    }

    ;; --- getter повідомлень ---
    if handle_getter_messages() {
        return (); 
    }

    
    ;; throw(error::wrong_op);
}

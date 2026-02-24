ZLK Jetton Minter
#include "stdlib.fc";
() recv_internal(int my_balance, int msg_value, cell in_msg_full, slice in_msg_body) impure {
    ;; перевірка що повідомлення має body
    throw_if(100, in_msg_body.slice_empty?());
ctx::init(my_balance, msg_value, in_msg_full, in_msg_body);
} 
if (ctx.at(IS_BOUNCED)) {
        return ();
    }

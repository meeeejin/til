# Buffer Manager

## Buffer fix

> *sm/bf_tree.cpp* 1580L: `w_rc_t bf_tree_m::fix()`

```cpp
w_rc_t bf_tree_m::fix(generic_page* parent, generic_page*& page,                                      
                                   PageID pid, latch_mode_t mode,                                     
                                   bool conditional, bool virgin_page,                                
                                   bool only_if_hit, bool do_recovery,                                
                                   lsn_t emlsn)                                                       
{
    ...
    while (true)                                                                                            
    {                                                                                                  
        bf_idx_pair p;                                                                                 
        bf_idx idx = 0;                                                                                
        if (_hashtable->lookup(pid, p)) {                                                              
            idx = p.first;                                                                             
            if (parent && p.second != parent - _buffer) {                                              
                // need to update parent pointer                                                       
                p.second = parent - _buffer;                                                           
                _hashtable->update(pid, p);                                                            
                INC_TSTAT(bf_fix_adjusted_parent);                                                     
            }                                                                                          
        }

        bool media_failure = is_media_failure(pid);                                                    
                                                                                                       
        if (idx == 0)                                                                                  
        {                                                                                              
            if (only_if_hit) {                                                                         
                return RC(stINUSE);                                                                    
            }                                                                                          
                                                                                                       
            // Wait for instant restore to restore this segment                                        
            if (do_recovery && !virgin_page && media_failure)                                          
            {                                                                                          
                // copy into local variable to avoid race condition with setting member to null        
                timer.reset();                                                                         
                auto restore = _restore_coord;                                                         
                if (restore) { restore->fetch(pid); }                                                  
                ADD_TSTAT(bf_batch_wait_time, timer.time_us());                                        
            }                                                                                          
                                                                                                       
            // STEP 1) Grab a free frame to read into                                                  
            W_DO(_grab_free_block(idx));                                                               
            bf_tree_cb_t &cb = get_cb(idx);                                                            
                                                                                                       
            // STEP 2) Acquire EX latch before hash table insert, to make sure                         
            // nobody will access this page until we're done                                           
            w_rc_t check_rc = cb.latch().latch_acquire(LATCH_EX,                                       
                    timeout_t::WAIT_IMMEDIATE);                                                        
            if (check_rc.is_error())                                                                   
            {                                                                                          
                _add_free_block(idx);                                                                  
                continue;                                                                              
            }                                                                                          
                                                                                                       
            // Register the page on the hashtable atomically. This guarantees                          
            // that only one thread will attempt to read the page                                      
            bf_idx parent_idx = parent ? parent - _buffer : 0;                                         
            bool registered = _hashtable->insert_if_not_exists(pid,                                    
                    bf_idx_pair(idx, parent_idx));                                                     
            if (!registered) {                                                                         
                cb.latch().latch_release();                                                            
                _add_free_block(idx);                                                                  
                continue;                                                                              
            }                                                                                          
                                                                                                
            w_assert1(idx != parent_idx);
    ...
}
```
# Buffer Manager

## Buffer page 구조

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
            // Page miss                                                                                              
            ...                                                                               
            // 페이지를 읽어올 free block 할당                                                 
            W_DO(_grab_free_block(idx));                                                               
            bf_tree_cb_t &cb = get_cb(idx);                                                            
                                                                                                       
            // Hash table에 insert 하기 전에, 해당 block에 외부 접근을 막기 위한 EX latch 획득                                      
            w_rc_t check_rc = cb.latch().latch_acquire(LATCH_EX,                                       
                    timeout_t::WAIT_IMMEDIATE);                                                        
            if (check_rc.is_error())                                                                   
            {                                                                                          
                _add_free_block(idx);                                                                  
                continue;                                                                              
            }                                                                                          
                                                                                                       
            // Hash table에 atomically insert                               
            bf_idx parent_idx = parent ? parent - _buffer : 0;                                         
            bool registered = _hashtable->insert_if_not_exists(pid,                                    
                    bf_idx_pair(idx, parent_idx));                                                     
            if (!registered) {                                                                         
                cb.latch().latch_release();                                                            
                _add_free_block(idx);                                                                  
                continue;                                                                              
            }                                                                                          
                                                                                                
            w_assert1(idx != parent_idx);
            // 디스크로부터 해당 page read                                                                    
            page = &_buffer[idx];                                                                     
                                                                                                      
            if (!virgin_page && !_no_db_mode) {                                                       
                INC_TSTAT(bf_fix_nonroot_miss_count);                                                 
                                                                                                      
                if (parent && emlsn.is_null() && _maintain_emlsn) {                                   
                    // Get emlsn from parent                                                          
                    general_recordid_t recordid = find_page_id_slot(parent, pid);                     
                    btree_page_h parent_h;                                                            
                    parent_h.fix_nonbufferpool_page(parent);                                          
                    emlsn = parent_h.get_emlsn_general(recordid);                                     
                }                                                                                     
                                                                                                      
                bool from_backup = media_failure && !do_recovery;
                // 페이지 read                                     
                W_DO(_read_page(pid, cb, from_backup));
                // 페이지 초기화                                               
                cb.init(pid, page->lsn);                                                              
                if (from_backup) { cb.pin_for_restore(); }                                          
            }                                                                                         
            else {                                                                                    
                ...                                                                                  
            }
        }
        else                                                                                          
        {                                                                                             
            // Page hit
            INC_TSTAT(bf_hit_cnt);                                                                    
            _hit_cnt++;                                                                               
                                                             
            bf_tree_cb_t &cb = get_cb(idx);

            ...
        }

        return RCOK;
    }
    ...
}
```

### Free block 할당

> *sm/bf_tree.cpp* 1580L: `w_rc_t bf_tree_m::_grab_free_block()`
```cpp
w_rc_t bf_tree_m::_grab_free_block(bf_idx& ret)                                                        
{                                                                                                      
    ret = 0;                                                                                           
                                                                                                       
    using namespace std::chrono;                                                                       
    auto time1 = steady_clock::now();                                                                  
                                                                                                       
    while (true) {                                                                                     
        // 매번 free list lock 잡는 건 cost가 있으니, lock을 잡기 전에 free list의 길이 먼저 확인              
        if (_freelist_len > 0) {                                                                       
            CRITICAL_SECTION(cs, &_freelist_lock);                                                     
            if (_freelist_len > 0) { // lock 잡고 실제로 확인                       
                bf_idx idx = FREELIST_HEAD;                                                            
                DBG5(<< "Grabbing idx " << idx);                                                       
                w_assert1(_is_valid_idx(idx));                                                         
                w_assert1 (!get_cb(idx)._used);                                                        
                w_assert1 (get_cb(idx)._pin_cnt < 0);                                                  
                ret = idx;                                                                             
                                                                                                       
                --_freelist_len;                                                                       
                if (_freelist_len == 0) {                                                              
                    FREELIST_HEAD = 0;                                                                 
                } else {                                                                               
                    FREELIST_HEAD = _freelist[idx];                                                    
                    w_assert1 (FREELIST_HEAD > 0 && FREELIST_HEAD < _block_cnt);                       
                }                                                                                      
                DBG5(<< "New head " << FREELIST_HEAD);                                                 
                w_assert1(ret != FREELIST_HEAD);                                                       
                break;                                                                                 
            }                                                                                          
        }                         
                                                                                                       
        // Free list에 free block이 없음                                                            
        set_warmup_done();                                                                             
                                                                                                       
        // Eviction 필요                                                      
        if (_async_eviction) {                                                                                         
            _evictioner->wakeup(true);                                                                 
        }
        else {                                                                                         
            bool success = false;                                                                      
            bf_idx victim = 0;                                                                         
            while (!success) {                                                                         
                // Victim 선정
                victim = _evictioner->pick_victim();                                                   
                w_assert0(victim > 0);                                                                 
                // Victim eviction
                success = _evictioner->evict_one(victim);                                              
            }                                                                                          
            ret = victim;                                                                              
            break;                                                                                     
        }                                                                                              
    }                                                                                                  
                                                                                                       
    auto time2 = steady_clock::now();                                                                  
    ADD_TSTAT(bf_evict_duration, duration_cast<nanoseconds>(time2-time1).count());                     
                                                                                                       
    return RCOK;                                                                                       
}
```

### Victim page 선정 및 eviction

> *sm/page_evictioner.cpp* 364L: `page_evictioner_base::pick_victim()`

```cpp
bf_idx page_evictioner_base::pick_victim()                                                             
{                                                                                                      
    bool ignore_dirty = _write_elision || _no_db_mode;                                                 
                                                                                                       
     auto next_idx = [this]                                                                            
     {                                                                                                 
         if (_current_frame > _bufferpool->_block_cnt) {                                               
             // race condition here, but it's not a big deal                                           
             _current_frame = 1;                                                                       
         }                                                                                             
         return _random_pick ? get_random_idx() : _current_frame++;                                    
     };                                                                                                
                                                                                                       
     unsigned attempts = 0;                                                                            
     while(true) {                                                                                     
                                                                                                       
        if (should_exit()) return 0; // in bf_tree.h, 0 is never used, means null                      
                                                                                                       
        bf_idx idx = next_idx();                                                                       
        if (idx >= _bufferpool->_block_cnt || idx == 0) { idx = 1; }                                   
                                                                                                       
        attempts++;                                                                                    
        if (attempts >= _max_attempts) {                                                               
            W_FATAL_MSG(fcINTERNAL, << "Eviction got stuck!");                                         
        }                                                                                              
        else if (_wakeup_cleaner_attempts > 0 && attempts % _wakeup_cleaner_attempts == 0)             
        {                                                                                              
            _bufferpool->wakeup_cleaner();                                                             
        }                                                                                              
        else if (_clean_only_attempts > 0 && attempts >= _clean_only_attempts)                         
        {                                                                                              
            ignore_dirty = true;                                                                       
        }                                                                                              
                                                                                                       
        auto& cb = _bufferpool->get_cb(idx);                                                           
                                                                                                       
        if (!cb._used) {                                                                               
            continue;                                                                                  
        }

        if (cb.latch().held_by_me()) {                                                                 
            // I (this thread) currently have the latch on this frame, so                              
            // obviously I should not evict it                                                         
            continue;                                                                                  
        }                                                                                              
                                                                                                       
        // EX mode로 latch 획득 
        rc_t latch_rc;                                                                                 
        latch_rc = cb.latch().latch_acquire(LATCH_EX, timeout_t::WAIT_IMMEDIATE);                      
        if (latch_rc.is_error()) {                                                                     
            DBG3(<< "Eviction failed on latch for " << idx);                                           
            continue;                                                                                  
        }                                                                                              
        w_assert1(cb.latch().is_mine());                                                               
                                                                                                       
        // Clock reference bit가 set 되어있는지 확인. Set 되어 있지 않아야 evict                                                       
        if (_use_clock && _clock_ref_bits[idx]) {                                                      
            _clock_ref_bits[idx] = false;                                                              
            cb.latch().latch_release();                                                                
            continue;                                                                                  
        }                                                                                              
                                                                                                       
        // eviction 가능한 페이지인지 확인                                                       
        btree_page_h p;                                                                                
        p.fix_nonbufferpool_page(_bufferpool->_buffer + idx);
        
        if (
                // ... the stnode page                                                                 
                p.tag() == t_stnode_p                                         
                // ... B-tree root pages                                                               
                // (note, single-node B-tree is both root and leaf)                                    
                || (p.tag() == t_btree_p && p.pid() == p.root())                                                                           
                // ... dirty pages, unless we're told to ignore them                                   
                || (!ignore_dirty && cb.is_dirty())                                                    
                // ... unused frames, which don't hold a valid page                                    
                || !cb._used                                                                           
                // ... pinned frames, i.e., someone required it not be evicted                         
                || cb._pin_cnt != 0                                                                    
                // ... frames prefetched by restore but not yet restored                               
                || cb.is_pinned_for_restore()                                                          
        )                                                                                              
        {                                                                                              
            cb.latch().latch_release();                                                                
            DBG5(<< "Eviction failed on flags for " << idx);                                           
            continue;                                                                                  
        }                                                                                              
                                                                                                       
        // 여기까지 왔으면, 해당 페이지 victim으로 선정                                      
        w_assert1(_bufferpool->_is_active_idx(idx));                                                   
        w_assert0(idx != 0);                                                                           
        ADD_TSTAT(bf_eviction_attempts, attempts);                                                     
        return idx;                                                                                    
    }                                                                                                  
}
```

> *sm/page_evictioner.cpp* 364L: `page_evictioner_base::evict_one()`
```cpp
bool page_evictioner_base::evict_one(bf_idx victim)                                                    
{                                                                                                      
    bf_tree_cb_t& cb = _bufferpool->get_cb(victim);                                                    
    w_assert1(cb.latch().is_mine());

    lsn_t page_lsn = cb.get_page_lsn();                                                                
    bool media_failure = _bufferpool->is_media_failure(cb._pid);                                       
    if (_no_db_mode || _write_elision || cb.is_dirty() || media_failure) {                                                                                         
        if (!page_lsn.is_null()) {                                                                     
            smlevel_0::recovery->add_dirty_page(cb._pid, page_lsn);                                    
        }                                                                                              
    }
    
    // Victim 페이지가 flush 돼야 하는지 확인 (dirty or not)                                                              
    bool was_dirty = cb.is_dirty();
    // Dirty page flush                                                                    
    if (was_dirty && !_write_elision) { flush_dirty_page(cb); }                                        
    w_assert1(cb.latch().is_mine());

    if (_log_evictions) {                                                                              
        Logger::log_sys<evict_page_log>(cb._pid, was_dirty, page_lsn);                                 
    }                                                                                                  
                                                                                                       
    // Hash table에서 해당 페이지 제거                                                                        
    w_assert1(cb._pin_cnt < 0);                                                                        
    w_assert1(!cb._used);                                                                              
    bool removed = _bufferpool->_hashtable->remove(cb._pid);                                           
    w_assert1(removed);
    
    cb.latch().latch_release();
    
    INC_TSTAT(bf_evict);                                                                               
    return true;                                                                                       
}
```
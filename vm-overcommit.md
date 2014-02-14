
CommitLimit と Committed_AS を seq_printf してる部分

```c
static int meminfo_proc_show(struct seq_file *m, void *v)
{
	committed = percpu_counter_read_positive(&vm_Committed_A);
	allowed = ((totalram_pages - hugetlb_total_pages())
		* sysctl_overcommit_ratio / 100) + total_swap_pages;

	seq_printf(m,
        "CommitLimit:    %8lu kB\n"
		"Committed_AS:   %8lu kB\n"

// ...

		K(allowed),
		K(committed),
```

 * CommitLimit = allowd は `(ページ数 - HUGEページ数) * vm.overcommmit_ratio /100 + swap のページ数` でよくある説明通り
   * HUGEページを引いているのが違うかな

## Committed_AS

 * `struct percpu_counter`

 * `extern struct percpu_counter vm_committed_as;`
 
vm_acct_memory, vm_unacct_memory で加減される

```
static inline void vm_acct_memory(long pages)
{
	percpu_counter_add(&vm_committed_as, pages);
}
```

```
static inline void vm_unacct_memory(long pages)
{
	vm_acct_memory(-pages);
}
```

security_vm_enough_memory(len)

# backlog あれこれ

下記の設定がどんなレイヤで作用するのかを追っている

 * net.core.somaxconn
 * net.core.netdev_max_backlog
 * net.ipv4.tcp_max_syn_backlog

http://d.hatena.ne.jp/nyant/20111216/1324043063 も詳しい

## メモ書き 
 
 * net.core.*
   * プロトコルに依存しない設定
 * net.ipv4.*

## net.core.somaxconn

カーネル では core.sysctl_somaxconn で定義されている

```c
/* init_net はデフォルトの netns */
static struct ctl_table netns_core_table[] = {
	{
		.ctl_name	= NET_CORE_SOMAXCONN,
		.procname	= "somaxconn",
		.data		= &init_net.core.sysctl_somaxconn,
		.maxlen		= sizeof(int),
		.mode		= 0644,
		.proc_handler	= proc_dointvec
	},
	{ .ctl_name = 0 }
};
```

SOMAXCONN は 128 がデフォルト値である。

```c
/* Maximum queue length specifiable by listen.  */
#define SOMAXCONN	128
```

backlog のサイズは listen(2) で指定できる

```c
int listen(int sockfd, int backlog);
```

が、backlog のサイズは listen(2) する際に core.sysctl_somaxconn に切り詰められる

```c
/*
 *	Perform a listen. Basically, we allow the protocol to do anything
 *	necessary for a listen, and if that works, we mark the socket as
 *	ready for listening.
 */

SYSCALL_DEFINE2(listen, int, fd, int, backlog)
{
	struct socket *sock;
	int err, fput_needed;
	int somaxconn;

	sock = sockfd_lookup_light(fd, &err, &fput_needed);
	if (sock) {
                /* netns の変換 */
		somaxconn = sock_net(sock->sk)->core.sysctl_somaxconn;
		if ((unsigned)backlog > somaxconn)
			backlog = somaxconn;

		err = security_socket_listen(sock, backlog);
		if (!err)
			err = sock->ops->listen(sock, backlog);

		fput_light(sock->file, fput_needed);
	}
	return err;
}
```

backlog がどのように扱われるかは プロトコルファミリと通信方式に依る

 * IPv4 + AF_INET + SOCK_STREAM なら sock->ops->listen は は inet_listen 呼び出しになる
   * backlog が `sk->sk_max_ack_backlog` としてセットされる
   * ACK を受け取れるキューのサイズ?

```
/*
 *	Move a socket into listening state.
 */
int inet_listen(struct socket *sock, int backlog)
{
	struct sock *sk = sock->sk;
	unsigned char old_state;
	int err;

	lock_sock(sk);

	err = -EINVAL;
	if (sock->state != SS_UNCONNECTED || sock->type != SOCK_STREAM)
		goto out;

	old_state = sk->sk_state;
	if (!((1 << old_state) & (TCPF_CLOSE | TCPF_LISTEN)))
		goto out;

	/* Really, if the socket is already in listen state
	 * we can only allow the backlog to be adjusted.
	 */
	if (old_state != TCP_LISTEN) {
		err = inet_csk_listen_start(sk, backlog);
		if (err)
			goto out;
	}
	sk->sk_max_ack_backlog = backlog;
	err = 0;

out:
	release_sock(sk);
	return err;
}
EXPORT_SYMBOL(inet_listen);
```

baclkog は nr_table_entries として reqsk_queue_alloc に渡される

```c
// nr_table_entries = backlog
int inet_csk_listen_start(struct sock *sk, const int nr_table_entries)
{
	struct inet_sock *inet = inet_sk(sk);
	struct inet_connection_sock *icsk = inet_csk(sk);
	int rc = reqsk_queue_alloc(&icsk->icsk_accept_queue, nr_table_entries);

	if (rc != 0)
		return rc;

	sk->sk_max_ack_backlog = 0;
	sk->sk_ack_backlog = 0;
	inet_csk_delack_init(sk);

	/* There is race window here: we announce ourselves listening,
	 * but this transition is still not validated by get_port().
	 * It is OK, because this socket enters to hash table only
	 * after validation is complete.
	 */
	sk->sk_state = TCP_LISTEN;
	if (!sk->sk_prot->get_port(sk, inet->num)) {
		inet->sport = htons(inet->num);

		sk_dst_reset(sk);
		sk->sk_prot->hash(sk);

		return 0;
	}

	sk->sk_state = TCP_CLOSE;
	__reqsk_queue_destroy(&icsk->icsk_accept_queue);
	return -EADDRINUSE;
}

EXPORT_SYMBOL_GPL(inet_csk_listen_start);
```

reqsk_queue_alloc で nr_table_entries は下記の様に扱われている

 * nr_table_entries (= backlog) と sysctl_max_syn_backlog の min()
 * nr_table_entries (= backlog) と 8 の max()
 * roundup_pow_of_two で 2の倍数に切り詰められる?
   * つまり `8 <= nr_table_entries <= sysctl_max_syn_backlog` に設定される
 * backlog 分の resquet_sock を vmalloc/kzalloc で割り当てる
   * = request_sock がメモリ上のデータとしての backlog の正体?

```c

//                     |-- nr_table_entries = backlog --|
//                     |                                |
//                     |                                |
// [listen_sock][request_sock][request_sock] ... [request_sock]
//    \
//     \
//  LISTEN の socket?
//

/*
 * Maximum number of SYN_RECV sockets in queue per LISTEN socket.
 * One SYN_RECV socket costs about 80bytes on a 32bit machine.
 * It would be better to replace it with a global counter for all sockets
 * but then some measure against one socket starving all other sockets
 * would be needed.
 *
 * The minimum value of it is 128. Experiments with real servers show that
 * it is absolutely not enough even at 100conn/sec. 256 cures most
 * of problems.
 * This value is adjusted to 128 for low memory machines,
 * and it will increase in proportion to the memory of machine.
 * Note : Dont forget somaxconn that may limit backlog too.
 */
int sysctl_max_syn_backlog = 256;

int reqsk_queue_alloc(struct request_sock_queue *queue,
		      unsigned int nr_table_entries)
{
	size_t lopt_size = sizeof(struct listen_sock);
	struct listen_sock *lopt;

	nr_table_entries = min_t(u32, nr_table_entries, sysctl_max_syn_backlog);
	nr_table_entries = max_t(u32, nr_table_entries, 8);
	nr_table_entries = roundup_pow_of_two(nr_table_entries + 1);
	lopt_size += nr_table_entries * sizeof(struct request_sock *);
	if (lopt_size > PAGE_SIZE)
		lopt = __vmalloc(lopt_size,
			GFP_KERNEL | __GFP_HIGHMEM | __GFP_ZERO,
			PAGE_KERNEL);
	else
		lopt = kzalloc(lopt_size, GFP_KERNEL);
	if (lopt == NULL)
		return -ENOMEM;

	for (lopt->max_qlen_log = 3;
	     (1 << lopt->max_qlen_log) < nr_table_entries;
	     lopt->max_qlen_log++);

	get_random_bytes(&lopt->hash_rnd, sizeof(lopt->hash_rnd));
	rwlock_init(&queue->syn_wait_lock);
	queue->rskq_accept_head = NULL;
	lopt->nr_table_entries = nr_table_entries;

	write_lock_bh(&queue->syn_wait_lock);
	queue->listen_opt = lopt;
	write_unlock_bh(&queue->syn_wait_lock);

	return 0;
}
```

## sk_max_ack_backlog と比較してドロップされる箇所はどこ?

sk_acceptq_is_full の場合パケットが drop? される

```c
int tcp_v4_conn_request(struct sock *sk, struct sk_buff *skb)
{

	struct request_sock *req;

// ...

	/* TW buckets are converted to open requests without
	 * limitations, they conserve resources and peer is
	 * evidently real one.
	 */
	if (inet_csk_reqsk_queue_is_full(sk) && !isn) {
#ifdef CONFIG_SYN_COOKIES
		if (sysctl_tcp_syncookies) {
			want_cookie = 1;
		} else
#endif
		goto drop;
	}

	/* Accept backlog is full. If we have already queued enough
	 * of warm entries in syn queue, drop request. It is better than
	 * clogging syn queue with openreqs with exponentially increasing
	 * timeout.
	 */
	if (sk_acceptq_is_full(sk) && inet_csk_reqsk_queue_young(sk) > 1)
		goto drop;

	req = inet_reqsk_alloc(&tcp_request_sock_ops);
	if (!req)
		goto drop;
```

sk_acceptq_is_full の中身は下記の通り。ここで backlog の数値が意味を茄子

```c
static inline int sk_acceptq_is_full(struct sock *sk)
{
	return sk->sk_ack_backlog > sk->sk_max_ack_backlog;
}
```

tcp_v4_syn_recv_sock でも sk_acceptq_is_full する

```c
/*
 * The three way handshake has completed - we got a valid synack -
 * now create the new socket.
 */
struct sock *tcp_v4_syn_recv_sock(struct sock *sk, struct sk_buff *skb,
				  struct request_sock *req,
				  struct dst_entry *dst)
{
	struct inet_request_sock *ireq;
	struct inet_sock *newinet;
	struct tcp_sock *newtp;
	struct sock *newsk;
#ifdef CONFIG_TCP_MD5SIG
	struct tcp_md5sig_key *key;
#endif
	struct ip_options *inet_opt;

	if (sk_acceptq_is_full(sk))
		goto exit_overflow;
```

tcp_v4_syn_recv_sock, tcp_v4_conn_request が出てくるのは ↓

```c
const struct inet_connection_sock_af_ops ipv4_specific = {
	.queue_xmit	   = ip_queue_xmit,
	.send_check	   = tcp_v4_send_check,
	.rebuild_header	   = inet_sk_rebuild_header,
	.conn_request	   = tcp_v4_conn_request,
	.syn_recv_sock	   = tcp_v4_syn_recv_sock,
	.remember_stamp	   = tcp_v4_remember_stamp,
	.net_header_len	   = sizeof(struct iphdr),
	.setsockopt	   = ip_setsockopt,
	.getsockopt	   = ip_getsockopt,
	.addr2sockaddr	   = inet_csk_addr2sockaddr,
	.sockaddr_len	   = sizeof(struct sockaddr_in),
	.bind_conflict	   = inet_csk_bind_conflict,
#ifdef CONFIG_COMPAT
	.compat_setsockopt = compat_ip_setsockopt,
	.compat_getsockopt = compat_ip_getsockopt,
#endif
};
```

## 割り込みからどのようにプロトコルスタックを上っていくかをつらつらを書きだす

struct virtio_driver virtio_net_driver の .probe 登録

 * virtnet_probe(struct virtio_device *vdev)
 * netif_napi_add(dev, &vi->napi, virtnet_poll, napi_weight);
 * NAPI に virtnet_poll の登録

 * virtnet_poll(struct napi_struct *napi, int budget)
 * receive_buf(struct net_device *dev, void *buf, unsigned int len)
 * netif_receive_skb へ続く

a. device driver が netif_receive_skb 呼び出し

 * netif_receive_skb(struct sk_buff *skb)
 * __netif_receive_skb
   * ... deliver_skb に続く

b. device driver が netif_rx 呼び出し

 * netif_rx(struct sk_buff *skb)
 * enqueue_to_backlog(struct sk_buff *skb, int cpu, unsigned int *qtail)
   * CPU ごとのバックログに sk_buff を突っ込む?
     `queue->input_pkt_queue.qlen <= netdev_max_backlog` でドロップするか否かを見る
   * **netdev_max_backlog**
     * プロトコルに関係ないレイヤ( net.core ) での backlog
 * ____napi_schedule
 * __raise_softirq_irqoff(NET_RX_SOFTIRQ)

open_softirq(NET_RX_SOFTIRQ, net_rx_action);

 * net_rx_action
 * ???

struct softnet_data *queue の .backlog.poll 呼び出し

 * process_backlog
 * __netif_receive_skb
 * deliver_skb

struct packet_type pt->prev->func で .func 呼び出し

 * ip_rcv
 * ip_rcv_finish
 * dst_input
 * ip_local_deliver
 * ip_local_deliver_finish

struct net_protocol *ipprot の .handler 呼び出し

 * tcp_v4_rcv
 * tcp_v4_do_rcv
   * sk->sk_state によってディスパッチ
 * tcp_rcv_state_process
   * TCP_LISTEN + SYN パケットを送られた
 * icsk->icsk_af_ops->conn_request

struct inet_connection_sock_af_ops の .conn_request で

 * tcp_v4_conn_request
   * sk_acceptq_is_full(sk) で **sk_max_ack_backlog** と比較

## process_backlog

 * softirq
 * input_pkt_queue

![2014-05-08 18 06 30](https://cloud.githubusercontent.com/assets/172456/2913701/247285f6-d690-11e3-8487-8abccd4515ce.png)

```c
static int process_backlog(struct napi_struct *napi, int quota)
{
	int work = 0;
	struct softnet_data *sd = container_of(napi, struct softnet_data, backlog);
	unsigned long start_time = jiffies;

	napi->weight = weight_p;
	do {
		struct sk_buff *skb;

		spin_lock_irq(&sd->input_pkt_queue.lock);
		skb = __skb_dequeue(&sd->input_pkt_queue);
		if (!skb) {
			/*
			 * Inline a custom version of __napi_complete().
			 * only current cpu owns and manipulates this napi,
			 * and NAPI_STATE_SCHED is the only possible flag set on backlog.
			 * we can use a plain write instead of clear_bit(),
			 * and we dont need an smp_mb() memory barrier.
			 */
			list_del(&napi->poll_list);
			napi->state = 0;

			spin_unlock_irq(&sd->input_pkt_queue.lock);
			break;
		}
		incr_input_queue_head(sd);
		spin_unlock_irq(&sd->input_pkt_queue.lock);

		__netif_receive_skb(skb);
	} while (++work < quota && jiffies == start_time);

	return work;
}
```

----

 * net.ipv4.tcp_syncookies 
 * http://d.hatena.ne.jp/nyant/20111216/1324043063
 * TCP_DEFER_ACCEPT

```
server/listen.c

#ifdef APR_TCP_DEFER_ACCEPT
        rv = apr_socket_opt_set(s, APR_TCP_DEFER_ACCEPT, 1);
        if (rv != APR_SUCCESS && !APR_STATUS_IS_ENOTIMPL(rv)) {
            ap_log_perror(APLOG_MARK, APLOG_WARNING, rv, p,
                              "Failed to enable APR_TCP_DEFER_ACCEPT");
        }
#endif

srclib/apr/network_io/unix/sockopt.c

    case APR_TCP_DEFER_ACCEPT:
#if defined(TCP_DEFER_ACCEPT)
        if (apr_is_option_set(sock, APR_TCP_DEFER_ACCEPT) != on) {
            int optlevel = IPPROTO_TCP;
            int optname = TCP_DEFER_ACCEPT;

            if (setsockopt(sock->socketdes, optlevel, optname, 
                           (void *)&on, sizeof(int)) == -1) {
                return errno;
            }
            apr_set_option(sock, APR_TCP_DEFER_ACCEPT, on);
        }
#else


strut net

net->core.sysctl_somaxconn
```
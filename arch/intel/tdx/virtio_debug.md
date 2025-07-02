# `vq->priv` 的来历
```c
//drivers/block/virtio_blk.c
virtblk_probe()
-> init_vq(vblk)
   -> vblk->vqs = kmalloc_array(num_vqs, sizeof(*vblk->vqs), GFP_KERNEL)
   -> vqs = kmalloc_array(num_vqs, sizeof(*vqs), GFP_KERNEL);
   -> virtio_find_vqs(vdev, num_vqs, vqs, callbacks, names, &desc)
         //include/linux/virtio_config.h
      -> vdev->config->find_vqs(vdev, nvqs, vqs, callbacks, names, NULL, desc)
         //drivers/virtio/virtio_mmio.c
      => vm_find_vqs()
            for (i = 0; i < nvqs; ++i)
            -> vqs[i] = vm_setup_vq(vdev, queue_idx++, callbacks[i], names[i], ctx ? ctx[i] : false);
               -> info = kmalloc(sizeof(*info), GFP_KERNEL);
               -> vq = vring_create_virtqueue(index, num, VIRTIO_MMIO_VRING_ALIGN, vdev, true, true, ctx, notify, callback, name);
                  vq->priv = info; //priv 来自这里
                  info->vq = vq;
                  return vq;
      for (i = 0; i < num_vqs; i++)
         vblk->vqs[i].vq = vqs[i];
   //include/linux/blk-mq.h
-> vblk->disk = blk_mq_alloc_disk(&vblk->tag_set, vblk);
      //block/blk-mq.c
   -> __blk_mq_alloc_disk(set, queuedata, &__key)
      -> q = blk_mq_init_queue_data(set, queuedata)
         -> q = blk_alloc_queue(set->numa_node);
               //block/blk-core.c
            -> q = kmem_cache_alloc_node(blk_requestq_cachep, GFP_KERNEL | __GFP_ZERO, node_id);
            q->queuedata = queuedata; //q->queuedata 就是 vblk
            blk_mq_init_allocated_queue(set, q)
            -> xa_init(&q->hctx_table) //初始化硬件上下文表
            -> blk_mq_realloc_hw_ctxs(set, q)
                  for (i = 0; i < set->nr_hw_queues; i++)
                  -> blk_mq_alloc_and_init_hctx(set, q, i, node)
                     -> hctx = blk_mq_alloc_hctx(q, set, node)
                        -> hctx = kzalloc_node(sizeof(struct blk_mq_hw_ctx), gfp, node)
                           hctx->queue = q; //q 挂在硬件上下文的 .queue 域，vblk 则是 hctx.queue.queuedata
                           return hctx;
                     -> blk_mq_init_hctx(q, set, hctx, hctx_idx)
                        -> xa_insert(&q->hctx_table, hctx_idx, hctx, GFP_KERNEL) //将硬件上下文插入表中
            return q;
```

```c
//block/blk-mq.c
blk_mq_alloc_request_hctx()
   struct blk_mq_alloc_data data = {
      .q = q,
      ...
   }
-> data.hctx = xa_load(&q->hctx_table, hctx_idx); //硬件上下文表中取出上下文
-> rq = blk_mq_rq_ctx_init(&data, blk_mq_tags_from_data(&data), tag);
   -> rq->mq_hctx = hctx; //将硬件上下文和请求 struct request *rq 关联起来
```

```c
//block/blk-mq.c
blk_mq_submit_bio()
   hctx = rq->mq_hctx; //请求 struct request *rq 中取回硬件上下文
-> blk_mq_try_issue_directly(hctx, rq)
   -> __blk_mq_issue_directly(hctx, rq, true)
      -> q->mq_ops->queue_rq(hctx, &bd)
         //drivers/block/virtio_blk.c
      => virtio_queue_rq()
            struct virtio_blk *vblk = hctx->queue->queuedata; //取回 vblk
            //drivers/virtio/virtio_ring.c
         -> virtqueue_notify(vblk->vqs[qid].vq)
            -> vq->notify(_vq)
               //drivers/virtio/virtio_pci_common.c
            => vp_notify(struct virtqueue *vq)
                  //lib/iomap.c
               -> iowrite16(vq->index, (void __iomem *)vq->priv) //最开始的 info
                  -> IO_COND(addr, outw(val,port), writew(val, addr))
```
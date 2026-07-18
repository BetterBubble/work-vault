---
title: deploy-verify-raw-2026-07-17
type: note
permalink: tacticum/00-inbox/deploy-verify-raw-2026-07-17
---

# deploy-verify-raw-2026-07-17

Сырьё ночного verify (черновик). Канон — [[deploy-verify-result]].

## Форма прогона (восстановлена из /tmp/scratch_run_all.sh)
```
docker exec -e HELM_RAG2_EXACT_KEY_BOOST=true -e HELM_RAG2_NOISE_FLOOR=<τ> -e HELM_RAG2_NOISE_ACTION=drop \
  helm-helm-1 python -m helm.eval.rag2_eval --golden /tmp/golden_wide.json --k 10 --out <out.json>
```
Скрипт параллельного запуска: `/tmp/run_one.sh` + 3× nohup (τ=0.5/0.6/0.7). Golden пришлось `docker cp` в контейнер (после деплоя /tmp контейнера очистился).

## СТАРЫЙ код (baseline, из /tmp/eval_runs.log Run P)
```
recall@10=0.7033 mrr=0.4287 ndcg@10=0.497 no_answer=0.0594 lat=9663ms
key_lookup recall=0.46 | title_dense recall=0.94 | negative_ood recall=0.00 (undefined)
```

## НОВЫЙ код c2796a5 (парсинг /tmp/new_tau0{5,6,7}.json)
```
tau=0.5  recall@10=0.9733 mrr=0.6969 ndcg=0.7657 no_ans=0.0594 lat=8616ms
         key_lookup recall=1.0 no_ans=0.0 | title_dense recall=0.9467 no_ans=0.0 | negative_ood(n=40) no_ans=0.95
tau=0.6  recall@10=0.9733 mrr=0.7021 ndcg=0.7697 no_ans=0.0594 lat=8616ms
         key_lookup recall=1.0 no_ans=0.0 | title_dense recall=0.9467 no_ans=0.0 | negative_ood(n=40) no_ans=0.95
tau=0.7  recall@10=0.975  mrr=0.7067 ndcg=0.7736 no_ans=0.0609 lat=8616ms
         key_lookup recall=1.0 no_ans=0.0 | title_dense recall=0.95 no_ans=0.0 | negative_ood(n=40) no_ans=0.975
```

## effort_hint (сырые ответы MCP)
- push-уведомления в чате: {similar=20, coverage 20/20, active_days med=1.9 p25=0.2 p75=8.7, lead_time med=15.4 p25=6.4 p75=39.7}
- экспорт списка пользователей: {similar=20, coverage 20/20, active_days med=1.1 p25=0.4 p75=10.4, lead_time med=79.2 p25=38.1 p75=112.5}

## тулы
- api_registry_check «отозвать письмо»: found=false, registries bot 1.30.0(5)/clients 2.30.0(342)/integration 1.30.0(72), as_of 2026-07-17
- contract_check «отзыв письма в JUMP»: found=true, JUMP:messageRevoke, score 4, command_count 101

## Заметки метода
- negative_ood: expected_keys нет → retrieval_undefined=True → исключены из recall-scored (n_retrieval_scored=600). recall на них 0.0 = мусор; сигнал шум-фильтра = no_answer_rate на срезе negative_ood.
- Оффлайн floor-sweep невозможен: `_hit_records` (rag2_runner.py) сериализует key/source/confidence/below_noise_floor, но НЕ `pinned` → оффлайн-порог задропал бы иммунные exact-key.
- benign в логах: `blockers(IVAONE-*) не удался: No such file /app/data/iva/jira_issue_links.csv` — путь/mount не смонтирован (отдельный трек #4 ночного плана), на key/title/negative метрики не влияет (structural_coverage=0).
</content>

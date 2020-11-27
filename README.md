# Embeeding Search for IR
Here, we conduct the experiments using Siamese BERT-base model. The maximum query and passage lengths are set to 32 and 150 (not including special tokens) respectively. For each query (document), we also put \[CLS\] and \[Q\](\[D\]) in the beginning. For the queries no longer than length of 32, we pad them with \[MASK\] tokens.

Requirement
---
tensorflow-gpu, faiss-gpu, progressbar

Passage Re-ranking with TCT-ColBERT embeeding
---
Here, we use average pooling embedding with dimension 768 (with 32 bits) to represent each query and passage. For re-ranking, we currently use CPU for dot product computation.
```shell=bash
mkdir embeddings prediction
candidate_file=msmarco-passage/top1000.dev.tsv
answer_file=msmarco-passage/qrels.dev.small.tsv
topk=1000
data_type=32 # 16 or 32 bits for embedding storage
emb_path=embeddings
query_emb=$emb_path\/query_emb/queries.doc.dev00.tf
corpus_emb=$emb_path\/corpus_emb/msmarco0
result_file=./prediction/rerank_result
###############################################
result_file=./prediction/rerank_result
python3 rerank.py --candidate_file $candidate_file \
                 --topk $topk --data_type $data_type \
                 --query_emb_path $query_emb --corpus_emb_path $corpus_emb \
                 --id_to_query_path $id_to_query \
                 --result_file $result_file.tsv
# Evaluation
python3 ./convert_msmarco_to_trec_run.py --input_run $result_file.tsv --output_run $result_file.trec
python3 ./msmarco_eval.py \
 msmarco-passage/qrels.dev.small.tsv $result_file.tsv
./trec_eval.9.0.4/trec_eval -c -mrecall.1000 \
 $qerl_file $result_file.trec
```
We do not consider the time for query embedding generation (7ms/query) into latency computation here.
Results  | Dev
------------| :------:
MRR10            | 0.3316
Recall@1000      | 0.8140
Latency (s/query)| 0.0070

End to End Passage Retrieval with TCT-ColBERT embeeding.
---
Indexing all MSMARCO passages in a file (Exhuasive search) requires 26 GB. For example, if only 4GB GPU is available for search, you can set max_passage_each_index to 1000,000 and 8 indexing files will be generated. Then, we search each index for topk passages, and merge and sort them to get the final ranking result.
```shell=bash
# First Store your query and passage embeddings tf record in embeddings/query_emb and embeddings/corpus_emb respectively.
mkdir prediction indexes
qerl_file=msmarco-passage/qrels.dev.small.tsv
topk=1000
num_files=10 # We currently save 1000,000 passage embeddings in each tf record file; thus total file number for corpus is 10.
max_passage_each_index=10,000,000
num_index=0 # It depends on max_passage_each_index: num_index=8.8M/max_passage_each_index
data_type=32 # 16 or 32 bits for embedding storage
corpus_type=passage #doc
index_file=indexes/msmarco_$corpus_type
gpu_device=0
emb_path=embeddings
query_emb=$emb_path\/query_emb/queries.doc.dev00.tf
corpus_emb=$emb_path\/corpus_emb/msmarco0
first_stage_path=first_stage_result
result_file=./prediction/rank_result
###############################################
# indexing (quantize option is on going)
python3 index.py --num_files $num_files --index_file $index_file \
     --topk $topk --corpus_emb_path $corpus_emb --data_type $data_type \
     --corpus_type $corpus_type --max_passage_each_index $max_passage_each_index \
# First-stage search with faiss
for i in {0..$num_index}
do
    python search.py --offset $i --index_file $index_file\_$i --pickle_file $first_stage_path/result\_$i.pickle\
        --topk $topk --query_emb_path $query_emb --data_type 32\
        --query_word_num $query_word_num --doc_word_num $doc_word_num --emb_dim $emb_dim --batch_size 1 --use_gpu \
        --gpu_device $gpu_device --passage_per_index $max_passage_each_index
done
# Output final result
python3 output_result.py --topk $topk --emb_path $emb_path --data_type $data_type --first_stage_path $first_stage_path\
                         --result_file $result_file.tsv \
                         --corpus_type $corpus_type \
                         --id_to_query_path $id_to_query \

# Evaluation
python3 ./convert_msmarco_to_trec_run.py --input_run $result_file.tsv --output_run $result_file.trec
python3 ./msmarco_eval.py \
 msmarco-passage/qrels.dev.small.tsv $result_file.tsv
./trec_eval.9.0.4/trec_eval -c -mrecall.1000 \
 $qerl_file $result_file.trec
```
Results  | Dev
------------| :------:
MRR10            | 0.3345
Recall@1000      | 0.9637
Latency (s/query)| 0.1000



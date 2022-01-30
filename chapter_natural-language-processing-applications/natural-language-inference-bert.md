# Doğal Dil Çıkarımı: İnce Ayar BERT
:label:`sec_natural-language-inference-bert`

Bu bölümün önceki bölümlerinde, SNLI veri kümesinde (:numref:`sec_natural-language-inference-and-dataset`'de açıklandığı gibi) doğal dil çıkarım görevi için dikkat tabanlı bir mimari (:numref:`sec_natural-language-inference-attention`'te) tasarladık. Şimdi BERT ince ayar yaparak bu görevi tekrar gözden geçiriyoruz. :numref:`sec_finetuning-bert`'da tartışıldığı gibi, doğal dil çıkarımı bir dizi düzeyi metin çifti sınıflandırma sorunudur ve ince ayar BERT yalnızca :numref:`fig_nlp-map-nli-bert`'te gösterildiği gibi ek bir MLP tabanlı mimari gerektirir. 

![This section feeds pretrained BERT to an MLP-based architecture for natural language inference.](../img/nlp-map-nli-bert.svg)
:label:`fig_nlp-map-nli-bert`

Bu bölümde, BERT'in önceden eğitilmiş küçük bir sürümünü indireceğiz, ardından SNLI veri kümesinde doğal dil çıkarımı için ince ayar yapacağız.

```{.python .input}
from d2l import mxnet as d2l
import json
import multiprocessing
from mxnet import gluon, np, npx
from mxnet.gluon import nn
import os

npx.set_np()
```

```{.python .input}
#@tab pytorch
from d2l import torch as d2l
import json
import multiprocessing
import torch
from torch import nn
import os
```

## Önceden Eğitimli BERT yükleniyor

Bert'i :numref:`sec_bert-dataset` ve :numref:`sec_bert-pretraining`'te WikiText-2 veri kümelerinde nasıl ön eğitileceğini açıkladık (orijinal BERT modelinin çok daha büyük corpora üzerinde önceden eğitildiğini unutmayın). :numref:`sec_bert-pretraining`'te tartışıldığı gibi, orijinal BERT modelinin yüz milyonlarca parametresi vardır. Aşağıda, önceden eğitilmiş BERT'in iki versiyonunu sunuyoruz: “bert.base” ince ayar yapmak için çok sayıda hesaplama kaynağı gerektiren orijinal BERT baz modeli kadar büyüktür, “bert.small” ise gösteriyi kolaylaştırmak için küçük bir versiyondur.

```{.python .input}
d2l.DATA_HUB['bert.base'] = (d2l.DATA_URL + 'bert.base.zip',
                             '7b3820b35da691042e5d34c0971ac3edbd80d3f4')
d2l.DATA_HUB['bert.small'] = (d2l.DATA_URL + 'bert.small.zip',
                              'a4e718a47137ccd1809c9107ab4f5edd317bae2c')
```

```{.python .input}
#@tab pytorch
d2l.DATA_HUB['bert.base'] = (d2l.DATA_URL + 'bert.base.torch.zip',
                             '225d66f04cae318b841a13d32af3acc165f253ac')
d2l.DATA_HUB['bert.small'] = (d2l.DATA_URL + 'bert.small.torch.zip',
                              'c72329e68a732bef0452e4b96a1c341c8910f81f')
```

Önceden eğitilmiş BERT modeli, sözcük dağarcığını tanımlayan bir “vocab.json” dosyası ve önceden eğitilmiş parametrelerin “önceden eğitilmiş.params” dosyasını içerir. Önceden eğitilmiş BERT parametrelerini yüklemek için aşağıdaki `load_pretrained_model` işlevini uyguluyoruz.

```{.python .input}
def load_pretrained_model(pretrained_model, num_hiddens, ffn_num_hiddens,
                          num_heads, num_layers, dropout, max_len, devices):
    data_dir = d2l.download_extract(pretrained_model)
    # Define an empty vocabulary to load the predefined vocabulary
    vocab = d2l.Vocab()
    vocab.idx_to_token = json.load(open(os.path.join(data_dir, 'vocab.json')))
    vocab.token_to_idx = {token: idx for idx, token in enumerate(
        vocab.idx_to_token)}
    bert = d2l.BERTModel(len(vocab), num_hiddens, ffn_num_hiddens, num_heads, 
                         num_layers, dropout, max_len)
    # Load pretrained BERT parameters
    bert.load_parameters(os.path.join(data_dir, 'pretrained.params'),
                         ctx=devices)
    return bert, vocab
```

```{.python .input}
#@tab pytorch
def load_pretrained_model(pretrained_model, num_hiddens, ffn_num_hiddens,
                          num_heads, num_layers, dropout, max_len, devices):
    data_dir = d2l.download_extract(pretrained_model)
    # Define an empty vocabulary to load the predefined vocabulary
    vocab = d2l.Vocab()
    vocab.idx_to_token = json.load(open(os.path.join(data_dir, 'vocab.json')))
    vocab.token_to_idx = {token: idx for idx, token in enumerate(
        vocab.idx_to_token)}
    bert = d2l.BERTModel(len(vocab), num_hiddens, norm_shape=[256],
                         ffn_num_input=256, ffn_num_hiddens=ffn_num_hiddens,
                         num_heads=4, num_layers=2, dropout=0.2,
                         max_len=max_len, key_size=256, query_size=256,
                         value_size=256, hid_in_features=256,
                         mlm_in_features=256, nsp_in_features=256)
    # Load pretrained BERT parameters
    bert.load_state_dict(torch.load(os.path.join(data_dir,
                                                 'pretrained.params')))
    return bert, vocab
```

Makinelerin çoğunda gösterimi kolaylaştırmak için, bu bölümde önceden eğitilmiş BERT'in küçük versiyonunu (“bert.small”) yükleyip ince ayar yapacağız. Egzersizde, test doğruluğunu önemli ölçüde artırmak için çok daha büyük “bert.base” in nasıl ince ayar yapılacağını göstereceğiz.

```{.python .input}
#@tab all
devices = d2l.try_all_gpus()
bert, vocab = load_pretrained_model(
    'bert.small', num_hiddens=256, ffn_num_hiddens=512, num_heads=4,
    num_layers=2, dropout=0.1, max_len=512, devices=devices)
```

## BERT ince ayar için Dataset

SNLI veri kümesinde aşağı akış görevi doğal dil çıkarımı için, özelleştirilmiş bir veri kümesi sınıfı `SNLIBERTDataset` tanımlıyoruz. Her örnekte, öncül ve hipotez bir çift metin dizisi oluşturur ve :numref:`fig_bert-two-seqs`'te tasvir edildiği gibi bir BERT girdi dizisine paketlenir. Hatırlayın :numref:`subsec_bert_input_rep` segment kimlikleri bir BERT giriş dizisinde öncül ve hipotezi ayırt etmek için kullanılır. BERT giriş dizisinin önceden tanımlanmış maksimum uzunluğu ile (`max_len`), giriş metni çiftinin uzununun son belirteci `max_len` karşılanıncaya kadar kaldırılmaya devam eder. BERT ince ayar için SNLI veri kümesinin oluşturulmasını hızlandırmak için, paralel olarak eğitim veya test örnekleri oluşturmak için 4 işçi süreci kullanıyoruz.

```{.python .input}
class SNLIBERTDataset(gluon.data.Dataset):
    def __init__(self, dataset, max_len, vocab=None):
        all_premise_hypothesis_tokens = [[
            p_tokens, h_tokens] for p_tokens, h_tokens in zip(
            *[d2l.tokenize([s.lower() for s in sentences])
              for sentences in dataset[:2]])]
        
        self.labels = np.array(dataset[2])
        self.vocab = vocab
        self.max_len = max_len
        (self.all_token_ids, self.all_segments,
         self.valid_lens) = self._preprocess(all_premise_hypothesis_tokens)
        print('read ' + str(len(self.all_token_ids)) + ' examples')

    def _preprocess(self, all_premise_hypothesis_tokens):
        pool = multiprocessing.Pool(4)  # Use 4 worker processes
        out = pool.map(self._mp_worker, all_premise_hypothesis_tokens)
        all_token_ids = [
            token_ids for token_ids, segments, valid_len in out]
        all_segments = [segments for token_ids, segments, valid_len in out]
        valid_lens = [valid_len for token_ids, segments, valid_len in out]
        return (np.array(all_token_ids, dtype='int32'),
                np.array(all_segments, dtype='int32'), 
                np.array(valid_lens))

    def _mp_worker(self, premise_hypothesis_tokens):
        p_tokens, h_tokens = premise_hypothesis_tokens
        self._truncate_pair_of_tokens(p_tokens, h_tokens)
        tokens, segments = d2l.get_tokens_and_segments(p_tokens, h_tokens)
        token_ids = self.vocab[tokens] + [self.vocab['<pad>']] \
                             * (self.max_len - len(tokens))
        segments = segments + [0] * (self.max_len - len(segments))
        valid_len = len(tokens)
        return token_ids, segments, valid_len

    def _truncate_pair_of_tokens(self, p_tokens, h_tokens):
        # Reserve slots for '<CLS>', '<SEP>', and '<SEP>' tokens for the BERT
        # input
        while len(p_tokens) + len(h_tokens) > self.max_len - 3:
            if len(p_tokens) > len(h_tokens):
                p_tokens.pop()
            else:
                h_tokens.pop()

    def __getitem__(self, idx):
        return (self.all_token_ids[idx], self.all_segments[idx],
                self.valid_lens[idx]), self.labels[idx]

    def __len__(self):
        return len(self.all_token_ids)
```

```{.python .input}
#@tab pytorch
class SNLIBERTDataset(torch.utils.data.Dataset):
    def __init__(self, dataset, max_len, vocab=None):
        all_premise_hypothesis_tokens = [[
            p_tokens, h_tokens] for p_tokens, h_tokens in zip(
            *[d2l.tokenize([s.lower() for s in sentences])
              for sentences in dataset[:2]])]
        
        self.labels = torch.tensor(dataset[2])
        self.vocab = vocab
        self.max_len = max_len
        (self.all_token_ids, self.all_segments,
         self.valid_lens) = self._preprocess(all_premise_hypothesis_tokens)
        print('read ' + str(len(self.all_token_ids)) + ' examples')

    def _preprocess(self, all_premise_hypothesis_tokens):
        pool = multiprocessing.Pool(4)  # Use 4 worker processes
        out = pool.map(self._mp_worker, all_premise_hypothesis_tokens)
        all_token_ids = [
            token_ids for token_ids, segments, valid_len in out]
        all_segments = [segments for token_ids, segments, valid_len in out]
        valid_lens = [valid_len for token_ids, segments, valid_len in out]
        return (torch.tensor(all_token_ids, dtype=torch.long),
                torch.tensor(all_segments, dtype=torch.long), 
                torch.tensor(valid_lens))

    def _mp_worker(self, premise_hypothesis_tokens):
        p_tokens, h_tokens = premise_hypothesis_tokens
        self._truncate_pair_of_tokens(p_tokens, h_tokens)
        tokens, segments = d2l.get_tokens_and_segments(p_tokens, h_tokens)
        token_ids = self.vocab[tokens] + [self.vocab['<pad>']] \
                             * (self.max_len - len(tokens))
        segments = segments + [0] * (self.max_len - len(segments))
        valid_len = len(tokens)
        return token_ids, segments, valid_len

    def _truncate_pair_of_tokens(self, p_tokens, h_tokens):
        # Reserve slots for '<CLS>', '<SEP>', and '<SEP>' tokens for the BERT
        # input
        while len(p_tokens) + len(h_tokens) > self.max_len - 3:
            if len(p_tokens) > len(h_tokens):
                p_tokens.pop()
            else:
                h_tokens.pop()

    def __getitem__(self, idx):
        return (self.all_token_ids[idx], self.all_segments[idx],
                self.valid_lens[idx]), self.labels[idx]

    def __len__(self):
        return len(self.all_token_ids)
```

SNLI veri kümesini indirdikten sonra, `SNLIBERTDataset` sınıfını başlatarak eğitim ve test örnekleri oluşturuyoruz. Bu tür örnekler, doğal dil çıkarımlarının eğitimi ve test edilmesi sırasında minibatch'larda okunacaktır.

```{.python .input}
# Reduce `batch_size` if there is an out of memory error. In the original BERT
# model, `max_len` = 512
batch_size, max_len, num_workers = 512, 128, d2l.get_dataloader_workers()
data_dir = d2l.download_extract('SNLI')
train_set = SNLIBERTDataset(d2l.read_snli(data_dir, True), max_len, vocab)
test_set = SNLIBERTDataset(d2l.read_snli(data_dir, False), max_len, vocab)
train_iter = gluon.data.DataLoader(train_set, batch_size, shuffle=True,
                                   num_workers=num_workers)
test_iter = gluon.data.DataLoader(test_set, batch_size,
                                  num_workers=num_workers)
```

```{.python .input}
#@tab pytorch
# Reduce `batch_size` if there is an out of memory error. In the original BERT
# model, `max_len` = 512
batch_size, max_len, num_workers = 512, 128, d2l.get_dataloader_workers()
data_dir = d2l.download_extract('SNLI')
train_set = SNLIBERTDataset(d2l.read_snli(data_dir, True), max_len, vocab)
test_set = SNLIBERTDataset(d2l.read_snli(data_dir, False), max_len, vocab)
train_iter = torch.utils.data.DataLoader(train_set, batch_size, shuffle=True,
                                   num_workers=num_workers)
test_iter = torch.utils.data.DataLoader(test_set, batch_size,
                                  num_workers=num_workers)
```

## İnce Ayar BERT

:numref:`fig_bert-two-seqs`'ün de belirttiği gibi, BERT doğal dil çıkarımı için ince ayar yalnızca iki tam bağlı katmandan oluşan ekstra bir MLP gerektirir (aşağıdaki `BERTClassifier` sınıfında `self.hidden` ve `self.output`). Bu MLP<cls>, hem öncül hem de hipotezin bilgilerini kodlayan özel “” belirtecinin BERT temsilini, doğal dil çıkarımının üç çıkışına dönüştürür: entailment, çelişki ve tarafsız.

```{.python .input}
class BERTClassifier(nn.Block):
    def __init__(self, bert):
        super(BERTClassifier, self).__init__()
        self.encoder = bert.encoder
        self.hidden = bert.hidden
        self.output = nn.Dense(3)

    def forward(self, inputs):
        tokens_X, segments_X, valid_lens_x = inputs
        encoded_X = self.encoder(tokens_X, segments_X, valid_lens_x)
        return self.output(self.hidden(encoded_X[:, 0, :]))
```

```{.python .input}
#@tab pytorch
class BERTClassifier(nn.Module):
    def __init__(self, bert):
        super(BERTClassifier, self).__init__()
        self.encoder = bert.encoder
        self.hidden = bert.hidden
        self.output = nn.Linear(256, 3)

    def forward(self, inputs):
        tokens_X, segments_X, valid_lens_x = inputs
        encoded_X = self.encoder(tokens_X, segments_X, valid_lens_x)
        return self.output(self.hidden(encoded_X[:, 0, :]))
```

Aşağıda, önceden eğitilmiş BERT modeli `bert`, aşağı akım uygulaması için `BERTClassifier` örneğine `net` beslenir. BERT ince ayarının ortak uygulamalarında, sadece ek MLP'nin (`net.output`) çıkış tabakasının parametreleri sıfırdan öğrenilecektir. Önceden eğitilmiş BERT kodlayıcının (`net.encoder`) ve ek MLP'nin gizli katmanının (`net.hidden`) tüm parametreleri ince ayarlanacaktır.

```{.python .input}
net = BERTClassifier(bert)
net.output.initialize(ctx=devices)
```

```{.python .input}
#@tab pytorch
net = BERTClassifier(bert)
```

:numref:`sec_bert`'te hem `MaskLM` sınıfının hem de `NextSentencePred` sınıfının çalıştıkları MLP'lerde parametrelere sahip olduğunu hatırlayın. Bu parametreler önceden eğitilmiş BERT modeli `bert` olanların bir parçasıdır ve `net` parametrelerinin dolayısıyla bir parçası. Bununla birlikte, bu tür parametreler sadece maskeli dil modelleme kaybını ve ön eğitim sırasında bir sonraki cümle tahmini kaybını hesaplamak içindir. Bu iki kayıp fonksiyonları aşağı akım uygulamalarına ince ayar ile alakasız, bu nedenle, BERT ince ayarlı olduğunda `MaskLM` ve `NextSentencePred` çalışan MLP'lerin parametreleri güncellenmez (durdu). 

Eski degradelere sahip parametrelere izin vermek için `ignore_stale_grad=True` bayrağı `d2l.train_batch_ch13` işlevinde `step` işlevinde ayarlanır. Bu işlevi, SNLI'nin eğitim setini (`train_iter`) ve test setini (`test_iter`) kullanarak `net` modelini eğitmek ve değerlendirmek için kullanıyoruz. Sınırlı hesaplama kaynakları nedeniyle, eğitim ve test doğruluğu daha da geliştirilebilir: tartışmalarını egzersizlerde bırakıyoruz.

```{.python .input}
lr, num_epochs = 1e-4, 5
trainer = gluon.Trainer(net.collect_params(), 'adam', {'learning_rate': lr})
loss = gluon.loss.SoftmaxCrossEntropyLoss()
d2l.train_ch13(net, train_iter, test_iter, loss, trainer, num_epochs, devices,
               d2l.split_batch_multi_inputs)
```

```{.python .input}
#@tab pytorch
lr, num_epochs = 1e-4, 5
trainer = torch.optim.Adam(net.parameters(), lr=lr)
loss = nn.CrossEntropyLoss(reduction='none')
d2l.train_ch13(net, train_iter, test_iter, loss, trainer, num_epochs, devices)
```

## Özet

* SNLI veri kümesindeki doğal dil çıkarımı gibi aşağı akım uygulamaları için önceden eğitilmiş BERT modelini ince ayar yapabiliriz.
* İnce ayar sırasında BERT modeli aşağı akım uygulaması için modelin bir parçası haline gelir. Sadece ön eğitim kaybı ile ilgili parametreler ince ayar sırasında güncellenmeyecektir. 

## Egzersizler

1. Hesaplama kaynağınız izin veriyorsa, orijinal BERT baz modeli kadar büyük olan çok daha büyük, önceden eğitilmiş BERT modeline ince ayar yapın. `load_pretrained_model` işlevinde bağımsız değişkenleri şu şekilde ayarlayın: 'bert.small' yerine 'bert.base' ile değiştirilmesi, sırasıyla `num_hiddens=256`, `ffn_num_hiddens=512`, `num_heads=4` ve `num_layers=2` ile 768, 3072, 12 ve 12 değerlerini artırın. İnce ayar zamanlarını artırarak (ve muhtemelen diğer hiperparametreleri ayarlayarak), 0.86'dan daha yüksek bir test doğruluğu elde edebilir misiniz?
1. Bir çift dizileri uzunluk oranlarına göre nasıl kesilir? Bu çift kesme yöntemini ve `SNLIBERTDataset` sınıfında kullanılan yöntemi karşılaştırın. Artıları ve eksileri nelerdir?

:begin_tab:`mxnet`
[Discussions](https://discuss.d2l.ai/t/397)
:end_tab:

:begin_tab:`pytorch`
[Discussions](https://discuss.d2l.ai/t/1526)
:end_tab:
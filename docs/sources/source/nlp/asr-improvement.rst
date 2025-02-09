Tutorial
===========================

In this tutorial we will train an ASR postprocessing model to correct mistakes in
output of end-to-end language model. This model method works similar to translation model
in contrast to traditional ASR language model rescoring. The model architecture is
attention based encoder-decoder where both encoder and decoder are initialized with
pretrained BERT language model. To train this model we collected dataset with typical
ASR errors by using pretrained Jasper ASR model :cite:`li2019jasper`.

Data
-----------
**Data collection.** We collected dataset for this tutorial with Jasper ASR model
:cite:`li2019jasper` trained on Librispeech dataset :cite:`panayotov2015librispeech`.
To download the Librispeech dataset, see :ref:`LibriSpeech_dataset`.
To obtain the pretrained Jasper model, see :ref:`Jasper_model`.
Librispeech training dataset consists of three parts -- train-clean-100, train-clean-360 and
train-clean-500 which give 281k training examples in total.
To augment this data we used two techniques:

* We split all training data into 10 folds and trained 10 Jasper models in cross-validation manner: a model was trained on 9 folds and used to make ASR predictions for the remaining fold.
* We took pretrained Jasper model and enabled dropout during inference on training data. This procedure was repeated multiple times with different random seeds.

**Data postprocessing.** The collecred dataset was postprocessed by removing duplicates
and examples with word error rate higher than 0.5.
The resulting training dataset consists of 1.7M pairs of "bad" English-"good" English examples.

**Dev and test datasets preparation**. Librispeech contains 2 dev datasets
(dev-clean and dev-other) and 2 test datasets (test-clean and test-other).
For our task we kept the same splits. We fed these datasets to a pretrained
Jasper model with the greedy decoding to get the ASR predictions that are used
for evaluation in our tutorial.

Importing parameters from pretrained BERT
-----------------------------------------
Both encoder and decoder are initialized with pretrained BERT parameters. Since BERT language
model has the same architecture as transformer encoder, there is no need to do anything
additional. To prepare decoder parameters from pretrained BERT we wrote a script
``get_decoder_params_from_bert.py`` that downloads BERT parameters from
pytorch-transformers repository :cite:`huggingface2019transformers` and maps them into a transformer decoder.
Encoder-decoder attention is initialized with self-attention parameters.
The script is located under ``nemo/scripts`` directory and accepts 2 arguments:
``--model_name`` (ex. ``bert-base-cased``, ``bert-base-uncased``, etc) and ``--save_to``
(a directory where the parameters will be saved):

    .. code-block:: bash

        $ python get_decoder_params_from_bert.py --model_name bert-base-uncased


Neural modules overview
--------------------------
First we define tokenizer to convert tokens into indices. We will use ``bert-base-uncased``
vocabukary, since our dataset only contains lower-case text:

    .. code-block:: python

        tokenizer = NemoBertTokenizer(pretrained_model="bert-base-uncased")


The encoder block is a neural module corresponding to BERT language model from
``nemo.nemo_nlp.huggingface`` collection:

    .. code-block:: python

        zeros_transform = neural_factory.get_module(
            name="ZerosLikeNM",
            params={},
            collection="nemo_nlp"
        )
        encoder = neural_factory.get_module(
            name="higgingface.BERT",
            params={
                "pretrained_model_name": args.pretrained_model_name,
                "local_rank": args.local_rank
            },
            collection="nemo_nlp"
        )

    .. tip::
        Making embedding size (as well as all other tensor dimensions) divisible
        by 8 will help to get the best GPU utilization and speed-up with mixed precision
        training.

We also pad the matrix of embedding parameters with zeros to have all the dimensions sizes
divisible by 8, which will speed up the computations on GPU with AMP:

    .. code-block:: python

        vocab_size = 8 * math.ceil(tokenizer.vocab_size / 8)
        tokens_to_add = vocab_size - tokenizer.vocab_size
        device = encoder.bert.embeddings.word_embeddings.weight.get_device()
        zeros = torch.zeros((tokens_to_add, args.d_model)).to(device=device)

        encoder.bert.embeddings.word_embeddings.weight.data = torch.cat(
            (encoder.bert.embeddings.word_embeddings.weight.data, zeros))


Next we construct transformer decoder neural module. Since we will be initializing decoder
with pretrained BERT parameters, we set hidden activation to ``"hidden_act": "gelu"`` and learn
positional encodings ``"learn_positional_encodings": True``:

    .. code-block:: python

        decoder = neural_factory.get_module(
            name="TransformerDecoderNM",
            params={
                "d_model": args.d_model,
                "d_inner": args.d_inner,
                "num_layers": args.num_layers,
                "num_attn_heads": args.num_heads,
                "ffn_dropout": args.ffn_dropout,
                "vocab_size": vocab_size,
                "max_seq_length": max_sequence_length,
                "embedding_dropout": args.embedding_dropout,
                "learn_positional_encodings": True,
                "hidden_act": "gelu",
                **dec_first_sublayer_params
            },
          collection="nemo_nlp"
          )

To load the pretrained parameters into decoder, we use ``restore_from`` attribute function
of the decoder neural module:

    .. code-block:: python

        decoder.restore_from(args.restore_from, local_rank=args.local_rank)


Model training
--------------

To train the model run ``asr_postprocessor.py.py`` located in ``nemo\examples\nlp`` directory.
We train with novograd optimizer :cite:`ginsburg2019stochastic`, learning rate ``lr=0.001``,
polynomial learning rate decay policy, ``1000`` warmup steps, per-gpu batch size of ``4096*8`` tokens,
and ``0.25`` dropout probability. We trained on 8 GPUS. To launch the training in
multi-gpu mode run the following command:

    .. code-block:: bash

        $ python -m torch.distributed.launch --nproc_per_node=8  asr_postprocessor.py --dataset_dir ../../tests/data/pred_real/ --restore_from ../../scripts/bert-base-uncased_decoder.pt



References
------------------

.. bibliography:: asr_impr.bib
    :style: plain
# POLARA
Polara is the first recommendation framework that allows a deeper analysis of recommender systems performance, based on the idea of feedback polarity (by analogy with sentiment polarity in NLP).

In addition to standard question of "how good a recommender system is at recommending relevant items", it allows assessing the ability of a recommender system to **avoid irrelevant recommendations** (thus, less likely to disappoint a user). You can read more about this idea in a reasearch paper [Fifty Shades of Ratings: How to Benefit from a Negative Feedback in Top-N Recommendations Tasks](http://arxiv.org/abs/1607.04228). The research results can be easily reproduced with this framework, visit a "fixed state" version of the code at https://github.com/Evfro/fifty-shades (there're also many usage examples).

The framework also features efficient tensor-based implementation of an algorithm, proposed in the paper, that takes full advantage of the polarity-based formulation. Currently, there is an [online demo](http://coremodel.azurewebsites.net) (for test purposes only), that demonstrates the effect of taking into account feedback polarity. 

## Usage example
A special effort was made to make a *recsys for humans*, which stresses on the ease of use of the framework. For example, that's how you build a pure SVD recommender on top of the [Movielens 1M](http://grouplens.org/datasets/movielens/) dataset:

```python
from polara.recommender.data import RecommenderData
from polara.recommender.models import SVDModel
from polara.tools.movielens import get_movielens_data

ml_data = get_movielens_data(get_genres=False)
data_model = RecommenderData(ml_data, 'userid', 'movieid', 'rating')
svd = SVDModel(data_model)
svd.build()
svd.evaluate()
```

## Creating new recommender models
Basic models can be extended by subclassing `RecommenderModel` class and defining two required methods: `self.build()` and `self.get_recommendations()`. Here's an example of a simple item-to-item recommender model:
```python
class CooccurenceModel(RecommenderModel):
    def __init__(self, *args, **kwargs):
        super(CooccurenceModel, self).__init__(*args, **kwargs)
        self.method = 'item-to-item' #pick some meaningful name
        
    def build(self):
        self._recommendations = None
        idx, val, shp = self.data.to_coo(tensor_mode=False)
        #np.ones_like makes feedback implicit
        user_item_matrix = sp.sparse.coo_matrix((np.ones_like(val), (idx[:, 0], idx[:, 1])),
                                          shape=shp, dtype=np.float64).tocsr()
        
        i2i_matrix = user_item_matrix.T.dot(user_item_matrix)
        #exclude "self-links"
        diag_vals = i2i_matrix.diagonal()
        i2i_matrix -= sp.sparse.dia_matrix((diag_vals, 0), shape=i2i_matrix.shape)
        self._i2i_matrix = i2i_matrix
        
    def get_recommendations(self):
        userid, itemid, feedback = self.data.fields
        test_data = self.data.test.testset
        i2i_matrix = self._i2i_matrix
        
        idx = (test_data[userid], test_data[itemid])
        val = np.ones_like(test_data[feedback]) #make feedback implicit
        shp = (idx[0].max()+1, i2i_matrix.shape[0])
        test_matrix = sp.sparse.coo_matrix((val, idx), shape=shp,
                                           dtype=np.float64).tocsr()
        i2i_scores = test_matrix.dot(self._i2i_matrix).A
        
        if self.filter_seen:
            #prevent seen items from appearing in recommendations
            self.downvote_seen_items(i2i_scores, idx)

        top_recs = self.get_topk_items(i2i_scores)
        return top_recs
```
An the model is ready for evaluation:
```python
i2i = CooccurenceModel(data_model)
i2i.build()
i2i.evaluate()
```

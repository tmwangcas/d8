# The `Dataset` API

```{.python .input  n=1}
#@save_all
import pathlib
import pandas as pd
from typing import Union, Sequence, Callable
import fnmatch
import numpy as np
import unittest

from d8 import base_dataset
from d8 import data_reader
```

```{.python .input  n=2}
class TabularDataset(base_dataset.BaseDataset):
    def __init__(self,
                 df: pd.DataFrame,
                 reader: data_reader.Reader,
                 label: str = -1,
                 name: str = '') -> None:
        super().__init__(df, reader, name)
        if isinstance(label, int):
            label = df.columns[label]
        if label not in df.columns:
            raise ValueError(f'Label {label} is not in {df.columns}')
        self.label = label

    @classmethod
    def from_csv(cls, data_path: Union[str, Sequence[str]], label, names=None, df_func=None) -> 'Dataset':
        header = 0 if names else 'infer'
        reader = cls.create_reader(data_path)
        filenames = [p.replace('#','/').replace('?select=','/').replace('+',' ').split('/')[-1]
                     for p in base_dataset.listify(data_path)]
        dfs = [pd.read_csv(reader.open(f), header=header, names=names) for f in filenames]
        df = dfs[0] if len(dfs) == 1 else pd.concat(dfs, axis=0, ignore_index=True)
        if df_func: df = df_func(df)
        return cls(df, reader, label)

    def summary(self):
        
        numeric_cols = len(self.df.drop(self.label, axis=1).select_dtypes('number').columns)
        return  pd.DataFrame([{'#examples':len(self.df),
                               '#numeric_features':numeric_cols,
                               '#category_features':len(self.df.columns) - 1 - numeric_cols,
                                'size(MB)':self.df.memory_usage().sum()/2**20,}])

class Dataset(TabularDataset):
    TYPE = 'tabular_classification'

    def summary(self):
        path = self._get_summary_path()
        if path and path.exists(): return pd.read_pickle(path)
        s = super().summary()
        s.insert(1, '#classes', self.df[self.label].nunique())
        if path and path.parent.exists(): s.to_pickle(path)
        return s
```

```{.json .output n=2}
[
 {
  "data": {
   "text/html": "<div>\n<style scoped>\n    .dataframe tbody tr th:only-of-type {\n        vertical-align: middle;\n    }\n\n    .dataframe tbody tr th {\n        vertical-align: top;\n    }\n\n    .dataframe thead th {\n        text-align: right;\n    }\n</style>\n<table border=\"1\" class=\"dataframe\">\n  <thead>\n    <tr style=\"text-align: right;\">\n      <th></th>\n      <th>#examples</th>\n      <th>#classes</th>\n      <th>#numeric_features</th>\n      <th>#category_features</th>\n      <th>size(MB)</th>\n    </tr>\n  </thead>\n  <tbody>\n    <tr>\n      <th>0</th>\n      <td>149</td>\n      <td>3</td>\n      <td>4</td>\n      <td>0</td>\n      <td>0.005806</td>\n    </tr>\n  </tbody>\n</table>\n</div>",
   "text/plain": "   #examples  #classes  #numeric_features  #category_features  size(MB)\n0        149         3                  4                   0  0.005806"
  },
  "execution_count": 2,
  "metadata": {},
  "output_type": "execute_result"
 }
]
```
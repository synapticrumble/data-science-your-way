
# Term Frequency - Inverse Document Frequency 101

Let's program here a basic and beautiful *Information Retrieval* concept such as
*[tf-idf](http://en.wikipedia.org/wiki/Tf%E2%80%93idf)*. In order to do so, we
will first define a basic in-memory search engine that allows to add documents
and search for them. The search results will contain relevant documents together
with the *tf-idf* value.


    from math import log
    import re
    
    class IrIndex:
        """An in-memory inverted index"""
        
        pattern = re.compile("^\s+|\s*,*\s*|\s+$")
        
        def __init__(self):
            self.index = {}
            self.documents = []
            self.tf = {}
        
        def index_document(self, document):
            ## split
            terms = [word for word in self.pattern.split(document)]
            
            ## add to documents
            self.documents.append(document)
            document_pos = len(self.documents)-1
            
            ## add posts to index, updating tf, idf
            for term in terms:
                if term not in self.index:
                    self.index[term] = []
                    self.tf[term] = []
                self.index[term].append(document_pos)
                self.tf[term].append(terms.count(term))
            
        
        def tf_idf(self, term):
            ## get tf for each document
            if term in self.tf:
                res = []
                for tf, post in zip(self.tf[term], self.index[term]):
                    idf = log( float( len(self.documents) ) / float( len(self.tf[term]) ) )
                    res.append((tf * idf, self.documents[post]))
                return res 
            else:
                return []
               

We create now our empty index.


    index = IrIndex()

Add some documents...


    index.index_document("Bruno Clair Chambertin Clos de Beze 2001, Bourgogne, France")
    index.index_document("Bruno Clair Chambertin Clos de Beze 2005, Bourgogne, France")
    index.index_document("Bruno Clair Clos Saint Jaques 2001, Bourgogne, France")
    index.index_document("Bruno Clair Clos Saint Jaques 2002, Bourgogne, France")
    index.index_document("Bruno Clair Clos Saint Jaques 2005, Bourgogne, France")
    index.index_document("Coche-Dury Bourgogne Chardonay 2005, Bourgogne, France")
    index.index_document("Chateau Margaux 1982, Bordeaux, France")
    index.index_document("Chateau Margaux 1996, Bordeaux, France")
    index.index_document("Chateau Latour 1982, Bordeaux, France")
    index.index_document("Domaine Raveneau Le Clos 2001, Bourgogne, France")

Let's try some terms. First, we search for a term that doesn't exists in any of
our documents.


    index.tf_idf("hello")




    []



Next, let's try a term that appears in few documents.


    index.tf_idf("Bordeaux")




    [(1.2039728043259361, 'Chateau Margaux 1982, Bordeaux, France'),
     (1.2039728043259361, 'Chateau Margaux 1996, Bordeaux, France'),
     (1.2039728043259361, 'Chateau Latour 1982, Bordeaux, France')]



A term with higher idf. That is, 'Margaux' is **less common** or **more
specific** than 'Bordeaux'. We see higher scores in general.


    index.tf_idf("Margaux")




    [(1.6094379124341003, 'Chateau Margaux 1982, Bordeaux, France'),
     (1.6094379124341003, 'Chateau Margaux 1996, Bordeaux, France')]



A term with tf higher than 1 for one of the documents. Now we search for
'Bourgogne', a **more common term** in our index, so we have **lower idf**. That
means lower scores in general but higher in cases with **higher tf** among them.


    index.tf_idf("Bourgogne")




    [(0.22314355131420976,
      'Bruno Clair Chambertin Clos de Beze 2001, Bourgogne, France'),
     (0.22314355131420976,
      'Bruno Clair Chambertin Clos de Beze 2005, Bourgogne, France'),
     (0.22314355131420976,
      'Bruno Clair Clos Saint Jaques 2001, Bourgogne, France'),
     (0.22314355131420976,
      'Bruno Clair Clos Saint Jaques 2002, Bourgogne, France'),
     (0.22314355131420976,
      'Bruno Clair Clos Saint Jaques 2005, Bourgogne, France'),
     (0.44628710262841953,
      'Coche-Dury Bourgogne Chardonay 2005, Bourgogne, France'),
     (0.44628710262841953,
      'Coche-Dury Bourgogne Chardonay 2005, Bourgogne, France'),
     (0.22314355131420976, 'Domaine Raveneau Le Clos 2001, Bourgogne, France')]



## Multi-term search

In order to do multi-term search, we will sum the tf-idf for each term per
document. In order to do so, we need a new `tf_idf` method for our index.


    def tf_idf_multi(self, terms):
        res = []
        hits = {}
        # sum tf-idfs for each hitting document
        hitting_terms = [term for term in self.pattern.split(terms) if term in self.tf]
        for term in hitting_terms: # for each term having at least on hit...
            for tf, post in zip(self.tf[term], self.index[term]): # store the tf-idf in hits for later sum
                if post not in hits:
                    hits[post] = []
                idf = log( float( len(self.documents) ) / float( len(self.tf[term]) ) )
                hits[post].append(tf * idf)
        # sum hits for each post
        for post in hits.iterkeys():
            tfidf = sum(hits[post])
            res.append((tfidf, self.documents[post]))
            
        return res 
    
    
    IrIndex.tf_idf_multi = tf_idf_multi

First, let's check that works the same for single term queries.


    index.tf_idf_multi("hello")




    []




    index.tf_idf_multi("Bordeaux")




    [(1.2039728043259361, 'Chateau Latour 1982, Bordeaux, France'),
     (1.2039728043259361, 'Chateau Margaux 1982, Bordeaux, France'),
     (1.2039728043259361, 'Chateau Margaux 1996, Bordeaux, France')]




    index.tf_idf_multi("Margaux")




    [(1.6094379124341003, 'Chateau Margaux 1982, Bordeaux, France'),
     (1.6094379124341003, 'Chateau Margaux 1996, Bordeaux, France')]




    index.tf_idf_multi("Bourgogne")




    [(0.22314355131420976,
      'Bruno Clair Chambertin Clos de Beze 2001, Bourgogne, France'),
     (0.22314355131420976,
      'Bruno Clair Chambertin Clos de Beze 2005, Bourgogne, France'),
     (0.22314355131420976,
      'Bruno Clair Clos Saint Jaques 2001, Bourgogne, France'),
     (0.22314355131420976,
      'Bruno Clair Clos Saint Jaques 2002, Bourgogne, France'),
     (0.22314355131420976,
      'Bruno Clair Clos Saint Jaques 2005, Bourgogne, France'),
     (0.8925742052568391,
      'Coche-Dury Bourgogne Chardonay 2005, Bourgogne, France'),
     (0.22314355131420976, 'Domaine Raveneau Le Clos 2001, Bourgogne, France')]



We try now with a multi term search where one of the terms doesn't hit any
document.


    index.tf_idf_multi("hello Bordeaux")




    [(1.2039728043259361, 'Chateau Latour 1982, Bordeaux, France'),
     (1.2039728043259361, 'Chateau Margaux 1982, Bordeaux, France'),
     (1.2039728043259361, 'Chateau Margaux 1996, Bordeaux, France')]



Multi-term, with disjoint results.


    index.tf_idf_multi("Bourgogne Bordeaux")




    [(0.22314355131420976,
      'Bruno Clair Chambertin Clos de Beze 2001, Bourgogne, France'),
     (0.22314355131420976,
      'Bruno Clair Chambertin Clos de Beze 2005, Bourgogne, France'),
     (0.22314355131420976,
      'Bruno Clair Clos Saint Jaques 2001, Bourgogne, France'),
     (0.22314355131420976,
      'Bruno Clair Clos Saint Jaques 2002, Bourgogne, France'),
     (0.22314355131420976,
      'Bruno Clair Clos Saint Jaques 2005, Bourgogne, France'),
     (0.8925742052568391,
      'Coche-Dury Bourgogne Chardonay 2005, Bourgogne, France'),
     (1.2039728043259361, 'Chateau Margaux 1982, Bordeaux, France'),
     (1.2039728043259361, 'Chateau Margaux 1996, Bordeaux, France'),
     (1.2039728043259361, 'Chateau Latour 1982, Bordeaux, France'),
     (0.22314355131420976, 'Domaine Raveneau Le Clos 2001, Bourgogne, France')]



And finally, a multi-term where some results have more than one term hitting. We
see how the score is increased.


    index.tf_idf_multi("Margaux Bordeaux")




    [(1.2039728043259361, 'Chateau Latour 1982, Bordeaux, France'),
     (2.8134107167600364, 'Chateau Margaux 1982, Bordeaux, France'),
     (2.8134107167600364, 'Chateau Margaux 1996, Bordeaux, France')]



With this we complete our introduction to the concept of Term Frequency -
Inverse Document Frequency.

#!/usr/bin/make -f
# vim:noet:ts=3
# $Id$
# @author Darlene Stewart based on previous Makefile by Samuel Larkin
# @brief Makefile for CLS Lexi-tech for building domain dev/test/train sets.
# 
# This Makefile can be used for processing: 
# English-French, Chinese-English, Danish-English. 
# It supports defining multiple domains or treating all TMX files a single domain.
#
# Typically, we need to create a train, one test, and two dev sets for each 
# domain. Dev and test sets are taken from each domain that will be used to tune
# a system, with the rest of the domain data going into the domain's train set.
# The two dev and one test sets are 1500 sentences each for big domains, or
# 1000 sentences for smaller domains.
# 
# Normally, the test set contains the most recent data, and the two dev sets
# contain the next most recent data from the corpus. It's also possible to
# create random sampled sub-domain dev and test sets.
#
# Technologies langagieres interactives / Interactive Language Technologies
# Tech. de l'information et des communications / Information and Communications Tech.
# Conseil national de recherches Canada / National Research Council Canada
# Copyright 2012, Sa Majeste la Reine du Chef du Canada
# Copyright 2012, Her Majesty in Right of Canada

include ../preparation/Makefile.params

.SECONDARY:
# Sets for a corpus cannot be made in parallel.
.NOTPARALLEL:

CORPORA_DIR ?= ${PREP_DIR}


vpath %.al ${CORPORA_DIR} .
vpath %.id ${CORPORA_DIR} .


# Help message to guide the user.
.PHONY: help
help:
	@echo "This Makefile will produce disjoint corpora sets."
	@echo "Generated corpora files will be of the form:"
	@echo "   <corpus>.<set>_<lang_id>.raw"
	@echo "Where:"
	@echo "   set     = {${ALL_SETS}}"
	@echo "   lang_id = {${SRC} ${TGT}}"
	@echo "   corpus  = {${ALL_CORPORA}}"
	@echo
	@echo "To build the corpora sets, type:"
	@echo "   make all"
	@echo "To re-build the files for an individual corpus, type: "
	@echo "   make clean.sets.<corpus>; make all"


.PHONY: all
#ifndef USE_RANDOM_SAMPLING
all: sets
#else
#all: random_sets
#endif


########################################
# Clean up
.PHONY: clean
clean: clean.inter $(addprefix clean.sets., ${ALL_CORPORA} all)

.PHONY: clean.inter
clean.inter:
	${RM} $(ALL_CORPORA:=_${SRC}.al) $(ALL_CORPORA:=_${TGT}.al) $(ALL_CORPORA:=.id)
	${RM} *.tmp*
	${RM} *.rtrain

.PHONY: $(addprefix clean.sets., ${ALL_CORPORA} all)
$(addprefix clean.sets., ${ALL_CORPORA} all): clean.sets.%:
	${RM} $(foreach s, ${ALL_SETS}, $*.$s*) $*${ET_AL}.${TRAIN_SET}* $*${ET_AL}.${ALL_SET}*


########################################
# Combine the extracted parallel corpora files into domain groupings
# and sort by creation date (in the .id file).
# Note: date info to sort on is in the second field of the id file.

.PHONY: combine
combine: $(foreach s, ${SFXS}, $(ALL_CORPORA:=$s))

ifeq ($(strip ${USE_RANDOM_SAMPLING}),)
   DO_SORT = LC_ALL=C sort -k2,2 -s
else
   DO_SORT = cat
endif

$(ALL_CORPORA:=.id): %.id:
	zcat -f $(addprefix ${CORPORA_DIR}/, $*_${SRC}.al) > $*_${SRC}.al
	zcat -f $(addprefix ${CORPORA_DIR}/, $*_${TGT}.al) > $*_${TGT}.al
	zcat -f $(addprefix ${CORPORA_DIR}/, $*.id) > $*.id
	paste $*.id $*_${SRC}.al $*_${TGT}.al | ${DO_SORT} > $*.db
	cut -f1 $*.db > $*.id
	cut -f2 $*.db > $*_${SRC}.al
	cut -f3 $*.db > $*_${TGT}.al
	${RM} $*.db

$(ALL_CORPORA:=_${SRC}.al): %_${SRC}.al: | %.id ;

$(ALL_CORPORA:=_${TGT}.al): %_${TGT}.al: | %.id ;

########################################	
# Create the domain train, dev, and test sets.
# 
# The test set contains the most recent data, while the dev sets contain the
# next most recent data from the corpus.
#
# It's also possible to create random sampled sub-domain dev and test sets
# by setting USE_RANDOM_SAMPLING=1.

.PHONY: sets

ifdef USE_DEFAULT_SET_NAMES
# USE_DEFAULT_SET_NAMES should be set only when creating a single set of corpus
# files when a corpus name stem in not to be included in the final set names.
sets: default_sets

DEFAULT_FILES = $(foreach s, ${DT_SETS}, $(addprefix $s, ${SFXS:.al=.raw})) \
                $(foreach s, ${TRAIN_SET} ${ALL_SET}, $(addprefix $s, ${SFXS:.al=.raw.gz}))

.PHONY: default_sets
default_sets: ${DEFAULT_FILES}

${DEFAULT_FILES}: %: corpus.%
	mv $< $@

else
# Normal path: corpus sets including corpus name stem in the set names.
sets: $(addprefix set., ${ALL_CORPORA})

.PHONY: $(addprefix set., ${ALL_CORPORA})
$(addprefix set., ${CORPORA_DT}): set.%: $(foreach s, ${DT_SETS}, $(addprefix %.$s, ${SFXS:.al=.raw}))
$(addprefix set., ${CORPORA_DT}): set.%: $(addprefix %.${TRAIN_SET}, ${SFXS:.al=.raw.gz})
$(addprefix set., ${CORPORA_DT}): set.%: $(addprefix %.${ALL_SET}, ${SFXS:.al=.raw.gz})

$(addprefix set., ${CORPORA_NO_DT}): set.%: $(addprefix %.${TRAIN_SET}, ${SFXS:.al=.raw.gz})
	ln -sf $*.${TRAIN_SET}_${SRC}.raw.gz $*.${ALL_SET}_${SRC}.raw.gz
	ln -sf $*.${TRAIN_SET}_${TGT}.raw.gz $*.${ALL_SET}_${TGT}.raw.gz
	ln -sf $*.${TRAIN_SET}.id $*.${ALL_SET}.id

endif

########################################	
# Extract the individual sets for the corpora.
# Normally:
#    Test set(s) is the most recent data.
#    Dev sets are the next most recent data (dev1 more recent than dev2).
# Alternatively (if USE_RANDOM_SAMPLING is set)
#    Test and dev sets are random sampled from the training data.
# In both cases:
#    Train set is the remainder, and
#    Train ${ET_AL} set is the remainder + any extra corpora for the domain
#    All set is train and test and dev (i.e. original corpora)
#    All ${ET_AL} set is train and test and dev + any extra corpora (i.e. all data for the domain)
# Upon request an overall all.train set is created which is the concatenation
# of all the train sets.

$(ALL_CORPORA:=.train_${SRC}.raw.tmp): %.train_${SRC}.raw.tmp: %_${SRC}.al
$(ALL_CORPORA:=.train_${TGT}.raw.tmp): %.train_${TGT}.raw.tmp: %_${TGT}.al
$(ALL_CORPORA:=.train.id.tmp): %.train.id.tmp: %.id
$(foreach c, ${ALL_CORPORA}, $c.train_${SRC}.raw.tmp $c.train_${TGT}.raw.tmp $c.train.id.tmp): %.tmp:
	cp $< $@

$(foreach c, ${CORPORA_DT}, $(foreach s, ${DT_SETS}, \
	$(eval $c.$s.id: $c.train_${SRC}.raw.tmp $c.train_${TGT}.raw.tmp $c.train.id.tmp) \
))

ifndef USE_RANDOM_SAMPLING
SET_SIZE ?= 1500

$(foreach c, ${CORPORA_BIG_DT}, $(addprefix $c., $(DT_SETS:=.id))): SET_SIZE=${SIZE.BIG}

$(foreach c, ${CORPORA_SMALL_DT}, $(addprefix $c., $(DT_SETS:=.id))): SET_SIZE=${SIZE.SMALL}

$(foreach c, ${CORPORA_DT}, $(addprefix $c., $(DT_SETS:=.id))): %.id:
	tail -n ${SET_SIZE} $(basename $*).train_${SRC}.raw.tmp > $*_${SRC}.raw
	tail -n ${SET_SIZE} $(basename $*).train_${TGT}.raw.tmp > $*_${TGT}.raw
	tail -n ${SET_SIZE} $(basename $*).train.id.tmp > $*.id
	head -n -${SET_SIZE} $(basename $*).train_${SRC}.raw.tmp > $(basename $*).train_${SRC}.raw.tmp2
	head -n -${SET_SIZE} $(basename $*).train_${TGT}.raw.tmp > $(basename $*).train_${TGT}.raw.tmp2
	head -n -${SET_SIZE} $(basename $*).train.id.tmp > $(basename $*).train.id.tmp2
	${MV} $(basename $*).train_${SRC}.raw.tmp2 $(basename $*).train_${SRC}.raw.tmp
	${MV} $(basename $*).train_${TGT}.raw.tmp2 $(basename $*).train_${TGT}.raw.tmp
	${MV} $(basename $*).train.id.tmp2 $(basename $*).train.id.tmp
else
RSAMPLE_CNT ?= 300
RSAMPLE_SIZE ?= 5

$(foreach c, ${CORPORA_BIG_DT}, $(addprefix $c., $(DT_SETS:=.id))): RSAMPLE_CNT=${RSAMPLE_CNT.BIG}

$(foreach c, ${CORPORA_SMALL_DT}, $(addprefix $c., $(DT_SETS:=.id))): RSAMPLE_CNT=${RSAMPLE_CNT.SMALL}

RSAMPLE_OPTIONS = -r -n ${RSAMPLE_CNT} -w ${RSAMPLE_SIZE}

$(foreach c, ${CORPORA_DT}, $(addprefix $c., $(DT_SETS:=.id))): %.id:
	sample_parallel_text ${RSAMPLE_OPTIONS} -s $(suffix $*) -seed ${SEED$(suffix $*)} $+
	${MV} $(basename $*).train_${SRC}.raw.tmp$(suffix $*) $*_${SRC}.raw
	${MV} $(basename $*).train_${TGT}.raw.tmp$(suffix $*) $*_${TGT}.raw
	${MV} $(basename $*).train.id.tmp$(suffix $*) $*.id
	sample_parallel_text -v ${RSAMPLE_OPTIONS} -s .tmp2 -seed ${SEED$(suffix $*)} $+
	${MV} $(basename $*).train_${SRC}.raw.tmp.tmp2 $(basename $*).train_${SRC}.raw.tmp
	${MV} $(basename $*).train_${TGT}.raw.tmp.tmp2 $(basename $*).train_${TGT}.raw.tmp
	${MV} $(basename $*).train.id.tmp.tmp2 $(basename $*).train.id.tmp
	touch $@
endif

$(foreach c, ${CORPORA_DT}, $(addprefix $c., $(DT_SETS:=_${SRC}.raw))): %_${SRC}.raw: | %.id ;
$(foreach c, ${CORPORA_DT}, $(addprefix $c., $(DT_SETS:=_${TGT}.raw))): %_${TGT}.raw: | %.id ;

$(foreach c, ${ALL_CORPORA}, \
	$(eval $c.${TRAIN_SET}.id: $(addprefix $c.train,_${SRC}.raw.tmp _${TGT}.raw.tmp .id.tmp)) \
	$(eval $c.${TRAIN_SET}.id: $(foreach x, ${$c.extra}, $(addprefix $x, ${SFXS}))) \
	$(eval $c.${ALL_SET}.id: $(addprefix $c,${SFXS})) \
	$(eval $c.${ALL_SET}.id: $(foreach x, ${$c.extra}, $(addprefix $x, ${SFXS}))) \
)

$(ALL_CORPORA:=.${TRAIN_SET}.id): %.${TRAIN_SET}.id:
	cat $*.train_${SRC}.raw.tmp | gzip > $*.${TRAIN_SET}_${SRC}.raw.gz
	cat $*.train_${TGT}.raw.tmp | gzip > $*.${TRAIN_SET}_${TGT}.raw.gz
	cat $*.train.id.tmp > $*.${TRAIN_SET}.id
	$(if ${$*.extra}, cat $*.train_${SRC}.raw.tmp $(filter %_${SRC}.al, $+) | gzip > $*${ET_AL}.${TRAIN_SET}_${SRC}.raw.gz)
	$(if ${$*.extra}, cat $*.train_${TGT}.raw.tmp $(filter %_${TGT}.al, $+) | gzip > $*${ET_AL}.${TRAIN_SET}_${TGT}.raw.gz)
	$(if ${$*.extra}, cat $*.train.id.tmp $(filter %.id, $+) > $*${ET_AL}.${TRAIN_SET}.id)
	rm $*.train_${SRC}.raw.tmp $*.train_${TGT}.raw.tmp $*.train.id.tmp

$(ALL_CORPORA:=.${TRAIN_SET}_${SRC}.raw.gz): %.${TRAIN_SET}_${SRC}.raw.gz: | %.${TRAIN_SET}.id ;
$(ALL_CORPORA:=.${TRAIN_SET}_${TGT}.raw.gz): %.${TRAIN_SET}_${TGT}.raw.gz: | %.${TRAIN_SET}.id ;

ifdef ET_AL
$(ALL_CORPORA:=${ET_AL}.${TRAIN_SET}.id): %${ET_AL}.${TRAIN_SET}.id: | %.${TRAIN_SET}.id ;
$(ALL_CORPORA:=${ET_AL}.${TRAIN_SET}_${SRC}.raw.gz): %${ET_AL}.${TRAIN_SET}_${SRC}.raw.gz: | %.${TRAIN_SET}.id ;
$(ALL_CORPORA:=${ET_AL}.${TRAIN_SET}_${TGT}.raw.gz): %${ET_AL}.${TRAIN_SET}_${TGT}.raw.gz: | %.${TRAIN_SET}.id ;
endif

$(ALL_CORPORA:=.${ALL_SET}.id): %.${ALL_SET}.id:
	cat $*_${SRC}.al | gzip > $*.${ALL_SET}_${SRC}.raw.gz
	cat $*_${TGT}.al | gzip > $*.${ALL_SET}_${TGT}.raw.gz
	cat $*.id > $*.${ALL_SET}.id
	$(if ${$*.extra}, cat $(filter %_${SRC}.al, $+) | gzip > $*${ET_AL}.${ALL_SET}_${SRC}.raw.gz)
	$(if ${$*.extra}, cat $(filter %_${TGT}.al, $+) | gzip > $*${ET_AL}.${ALL_SET}_${TGT}.raw.gz)
	$(if ${$*.extra}, cat $(filter %.id, $+) > $*${ET_AL}.${ALL_SET}.id)

$(ALL_CORPORA:=.${ALL_SET}_${SRC}.raw.gz): %.${ALL_SET}_${SRC}.raw.gz: | %.${ALL_SET}.id ;
$(ALL_CORPORA:=.${ALL_SET}_${TGT}.raw.gz): %.${ALL_SET}_${TGT}.raw.gz: | %.${ALL_SET}.id ;

ifdef ET_AL
$(ALL_CORPORA:=${ET_AL}.${ALL_SET}.id): %${ET_AL}.${ALL_SET}.id: | %.${ALL_SET}.id ;
$(ALL_CORPORA:=${ET_AL}.${ALL_SET}_${SRC}.raw.gz): %${ET_AL}.${ALL_SET}_${SRC}.raw.gz: | %.${ALL_SET}.id ;
$(ALL_CORPORA:=${ET_AL}.${ALL_SET}_${TGT}.raw.gz): %${ET_AL}.${ALL_SET}_${TGT}.raw.gz: | %.${ALL_SET}.id ;
endif

# If requested, also create an overall train set (all.train) by concatenating
# all the other train sets. Such a set is useful for training truecasing and/or
# distortion models.
ifdef DO_ALL.TRAIN_SET
sets: set.all

.PHONY: set.all
set.all: $(addprefix all.${TRAIN_SET}, ${SFXS:.al=.raw.gz})

all.${TRAIN_SET}.id: %.id: $(foreach c, ${ALL_CORPORA}, $(addprefix $c$(if ${$c.extra},${ET_AL}).${TRAIN_SET}, ${SFXS:.al=.raw.gz}))
	cat $(filter %_${SRC}.raw.gz, $+) > $*_${SRC}.raw.gz
	cat $(filter %_${TGT}.raw.gz, $+) > $*_${TGT}.raw.gz
	cat $(filter %.id, $+) > $*.id

all.${TRAIN_SET}_${SRC}.raw.gz: | all.${TRAIN_SET}.id ;
all.${TRAIN_SET}_${TGT}.raw.gz: | all.${TRAIN_SET}.id ;
endif


############################################################
# HELPERS
########################################
# Check the counts.
.PHONY: check.counts

check.counts: $(addprefix check.counts., ${ALL_CORPORA} ${EXTRA_CORPORA})

$(addprefix check.counts., ${CORPORA_DT}): check.counts.%:
	-wc -l $*_${SRC}.al $*_${TGT}.al $*.id
	-wc -l $*.*_${SRC}.raw
	-wc -l $*.*_${TGT}.raw
	-wc -l $*.*.id
	-zcat $*.${TRAIN_SET}_${SRC}.raw.gz | wc -l
	-zcat $*.${TRAIN_SET}_${TGT}.raw.gz | wc -l
	-wc -l $*.${TRAIN_SET}.id
	-zcat $*.${ALL_SET}_${SRC}.raw.gz | wc -l
	-zcat $*.${ALL_SET}_${TGT}.raw.gz | wc -l
	-wc -l $*.${ALL_SET}.id
	@echo

$(addprefix check.counts., ${CORPORA_NO_DT}): check.counts.%:
	-wc -l $*_${SRC}.al $*_${TGT}.al $*.id
	-zcat $*.${TRAIN_SET}_${SRC}.raw.gz | wc -l
	-zcat $*.${TRAIN_SET}_${TGT}.raw.gz | wc -l
	-wc -l $*.${TRAIN_SET}.id
	-zcat $*.${ALL_SET}_${SRC}.raw.gz | wc -l
	-zcat $*.${ALL_SET}_${TGT}.raw.gz | wc -l
	-wc -l $*.${ALL_SET}.id
	@echo

$(addprefix check.counts., ${EXTRA_CORPORA}): check.counts.%:
	-wc -l $(addprefix ${CORPORA_DIR}/,$(addsuffix _${SRC}.al,$($*)))
	-wc -l $(addprefix ${CORPORA_DIR}/,$(addsuffix _${TGT}.al,$($*)))
	-wc -l $(addprefix ${CORPORA_DIR}/,$(addsuffix .id,$($*)))

# Check the counts on just the .id files
.PHONY: check.counts.id
check.counts.id: $(addprefix check.counts.id., ${ALL_CORPORA} ${EXTRA_CORPORA})

$(addprefix check.counts.id., ${ALL_CORPORA}): check.counts.id.%:
	-wc -l $*.*.id
	@echo

$(addprefix check.counts.id., ${EXTRA_CORPORA}): check.counts.id.%:
	-wc -l $(addprefix ${CORPORA_DIR}/,$(addsuffix .id,$($*)))

########################################
# Count very long sentences (>200 words).
.PHONY: count.long

LONG ?= 200

count.long: $(addprefix count.long., $(foreach c,${ALL_CORPORA}, $(foreach s,${ALL_SETS}, $c.$s_${SRC}.raw $c.$s_${TGT}.raw)))

count.long.%: %
	@echo "$<: " `cat $< | \
	   perl -le 'my ( \$$len, \$$cnt, \$$longest) = (0, 0, 0); \
	             while (<>) { \
	                \$$len = scalar(split); \
	                ++\$$cnt if \$$len > ${LONG}; \
	                \$$longest = \$$len if \$$len > \$$longest; \
	             } \
	             print "longest: ", \$$longest, ", > ${LONG}: ", \$$cnt ;'`

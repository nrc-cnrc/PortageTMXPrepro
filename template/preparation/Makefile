#!/usr/bin/make -f
# vim:noet:ts=3
# @author Darlene Stewart based on previous Makefile by Samuel Larkin
# @brief Makefile for CLS Lexi-tech for extracting parallel corpora from TMX files.
# 
# This Makefile can be used for processing: 
# English-French, Chinese-English, Danish-English. 
# It supports extraction from multiple TMX files.
#
# Technologies langagieres interactives / Interactive Language Technologies
# Tech. de l'information et des communications / Information and Communications Tech.
# Conseil national de recherches Canada / National Research Council Canada
# Copyright 2012, Sa Majeste la Reine du Chef du Canada
# Copyright 2012, Her Majesty in Right of Canada

include ./Makefile.params

.SECONDARY:

CORPORA_DIR ?= ${TMX_DIR}

CORPORA_FILES = $(foreach c, ${ORIG_CORPORA}, $(${c}))

vpath %.tmx . ${CORPORA_DIR}

# Sort by doc id?
SORT_BY_ID ?= $(if $(strip ${USE_RANDOM_SAMPLING}),1)


.PHONY: all
all: tu


########################################
# Clean up
.PHONY: clean clean.inter clean.all
clean: clean.inter
	${RM} $(CORPORA_FILES:=.utf8.tmx)
	${RM} $(CORPORA_FILES:=_${SRC}.al) $(CORPORA_FILES:=_${TGT}.al) $(CORPORA_FILES:=.id)
	
clean.inter:
	${RM} $(CORPORA_FILES:=.${TMX_SRC}) $(CORPORA_FILES:=.${TMX_TGT})
	${RM} $(CORPORA_FILES:=.${TMX_SRC}${TXTX}) $(CORPORA_FILES:=.${TMX_TGT}${TXTX})
	${RM} $(CORPORA_FILES:=.${TMX_SRC}.clean) $(CORPORA_FILES:=.${TMX_TGT}.clean)
	${RM} $(CORPORA_FILES:=.${TMX_SRC}${TXTX}.clean) $(CORPORA_FILES:=.${TMX_TGT}${TXTX}.clean)
	${RM} $(CORPORA_FILES:=.db) $(CORPORA_FILES:=.id.tmp)
	${RM} $(CORPORA_FILES:=.db.*sorted)
	${RM} $(CORPORA_FILES:=.utf8.clean.tmx)
#	${RM} $(CORPORA_FILES:=.utf8.tmx)

clean.links:
	${RM} $(CORPORA_FILES:=.tmx)

clean.all: clean clean.links

########################################
# Link to original corpora files

links: SHELL=/bin/bash
links:
	@for f in ${CORPORA_DIR}/*.tmx; do \
	   f2=`basename "$${f// /_}"`; \
	   f2=$${f2//(/_}; \
	   f2=$${f2//)/}; \
	   echo ln -s -f \"$$f\" $$f2; \
	   ln -s -f "$$f" $$f2; \
	done


########################################
# Step 0: Convert orignal TMX files to "clean" utf-8 that tmx2lfl.pl can parse.
#UTF8 ?=
UTF8 ?= .utf8.clean

.PHONY: utf8
utf8: $(CORPORA_FILES:=.utf8.tmx)

# Convert the TMX file to utf-8, if necessary.
# Some TMX files contain illegal input sequences, so have iconv discard these
# (-c).
%.utf8.tmx: %.tmx
	head -c 10 $< | xxd
	@if head -c 10 $< | xxd | grep '<?xml'; then \
	   echo Looks like $< is already in utf-8, filtering it instead of converting it; \
	   cat $< \
	   | iconv -c -f UTF-8 -t UTF-8 \
	   > $@; \
	else \
	   echo Assuming $< is in utf-16, converting to utf-8; \
	   cat $< \
	   | iconv -c -f UTF-16 -t UTF-8 \
	   > $@; \
	fi

# Some TMX files contain control characters (as literals or entities) that the
# XML parser used by tmx2lfl.pl cannot parse.
# \x0A (LF or \n) and \x0D (CR or \r) are OK, but all other characters from
# \x01 to \x1f must go.
#  - \x1e is replaced by hyphen because MSWord uses it as non-breaking hyphen
#  - \x1f is simply deleted because MSWord uses it as optional hyphen
#    (indicates where to break a word if it needs to be wrapped across two
#    lines), so it normally does not mark a word break.
#  - all others are replaced by a space, which will cause a token break, which
#    is expected to be the right thing to do most of the time.
%.utf8.clean.tmx: %.utf8.tmx
	cat $< \
	| perl -ple 'use encoding "UTF-8"; \
	             s/[\x01-\x09\x0B\x0C\x0E-\x1d]/ /g; \
	             s/[\x1e]/-/g; \
	             s/[\x1f]//g;  \
	             s/&#x[1-9bce];/ /ig; \
	             s/&#x1[0-9a-d];/ /ig; \
	             s/&#x1e;/-/ig; \
	             s/&#x1f;//ig; ' \
	> $@

# Make sure the TMX file is all valid utf8
.PHONY: validate_utf8
validate_utf8: $(addprefix validate., ${CORPORA_FILES})

validate.%: %.utf8.tmx
	utf8_filter -v -c < $< | egrep '\[__[[:xdigit:]]+\]' || true

# Validate that a file is valid utf8
validate.utf8.%: %
	utf8_filter -v -c < $< | egrep '\[__[[:xdigit:]]+\]' || true


########################################
# Extract the translation units.
.PHONY: tu
tu: $(CORPORA_FILES:=_${SRC}.al) $(CORPORA_FILES:=_${TGT}.al)

# Step 1: Extract bilingual corpora from TMX files.

ifdef TXTX
   TXT_OPT = -txt=${TXTX}
endif
%.${TMX_SRC}${TXTX} %.${TMX_TGT}${TXTX} %.id.tmp: %${UTF8}.tmx
	tmx2lfl.pl -verbose -timestamp -filename -output $* -src ${TMX_SRC} -tgt ${TMX_TGT} ${TXT_OPT} $<
	mv $*.id $*.id.tmp

# Step 2: Clean up and normalize the text.
#   - Convert wide punctuation character to space-surrounded normal ones in the EN-GB.
#   - Strip HTML markup where the HTML code starts with a non-uppercase character.
#   (Note: some TMX files contain some non-HTML text sequences surrounded by
#   <>, but these start with an uppercase letter whereas the HTML sequences are
#   lowercase.)
#   - Replace control characters that are invalid in plain text by spaces.
#   - Normalize spaces.
#   - Replace occurrences of ||| by ___|||___ since ||| is an invalid token in phrase tables.

FIX_WIDE_CHARS = s/([，。：）（；？﹗．﹪﹡﹟])/ \1 /g; tr/，。：）（；？﹗．﹪﹡﹟/,.:)(;?!.%*\#/;

ifneq ("$(filter ZH-CN,${TMX_SRC} ${TMX_TGT})","")
%.EN-GB${TXTX}.clean: DO_FIX_WIDE_CHARS = ${FIX_WIDE_CHARS}
%.EN-GB.clean: DO_FIX_WIDE_CHARS = ${FIX_WIDE_CHARS}
endif

# This expression replaces the official Unicode non-breaking hyphen (\x2011),
# MS Word's non-breaking hyphen (alt-0173, \xAD, officially the soft hyphen),
# MS Word's former non-breaking hyphen (\x1E), by a plain hyphen, -.
HYPHENS_TO_NORMALIZE = s/[‑­\x1E]/-/g;
# This expression removes MS Word's discretional hyphen, \x1F.
HYPHENS_TO_REMOVE = s/[\x1F]//g;
# Non breaking spaces
NON_BREAKING_SPACES := s/[\x{2060}\x{FEFF}\x{00A0}\x{2007}\x{202F}]/ /g;

%.clean: %
	cat $< \
	| perl -ple 'BEGIN{use encoding "UTF-8"; use utf8}; \
	             ${DO_FIX_WIDE_CHARS} \
	             ${HYPHENS_TO_NORMALIZE} \
	             ${HYPHENS_TO_REMOVE} \
	             s/\s*(<[^A-Z>][^>]*>\s*)+\s*/ /g; \
	             ${NON_BREAKING_SPACES} \
	             s/[\x01-\x09\x0B\x0C\x0E-\x1F\x7F]/ /g; \
	             s/\s+/ /g; \
	             s/(^| )\|\|\|(?= |$$)/ ___|||___/g; \
	             s/^\s+//g; \
	             s/\s+$$//g;' \
	> $@

# Step 3: Remove mono-lingual entries.
#   Remove any parallel entry that is empty in either file (paste cmd).
#   Group sentences per document (if SORT_BY_ID is defined).
#   Note: id spans two fields for BT-NB documents.

ifneq ($(strip ${SORT_BY_ID}),)
   DO_SORT = LC_ALL=C sort -k1,1 -s
else
   DO_SORT = cat
endif

# WARNING: Make sure your .clean files are tab-free.
%_${SRC}.al %_${TGT}.al %.id: %.${TMX_SRC}${TXTX}.clean %.${TMX_TGT}${TXTX}.clean %.id.tmp
	paste $*.id.tmp $*.${TMX_SRC}${TXTX}.clean $*.${TMX_TGT}${TXTX}.clean \
	| perl -ne 'print $$_ unless /(^|\t)(\t|$$)/;' \
	| ${DO_SORT} \
	> $*.db
	cut -f1 $*.db | perl -ple 's/\s+/ /g' > $*.id
	cut -f2 $*.db | perl -ple 's/\s+/ /g' > $*_${SRC}.al
	cut -f3 $*.db | perl -ple 's/\s+/ /g' > $*_${TGT}.al
	${RM} $*.db


########################################
# Let do some minimal validation on the output files.
.PHONY: validate
validate: SHELL=bash
validate: $(addprefix ${CORPORA}, _${SRC}.al _${TGT}.al .id)
	! egrep '^ *$$|EMPTY_' $+
	[[ "`wc -l < $(filter %_${SRC}.al, $+)`" == "`wc -l < $(filter %.id, $+)`" ]]
	[[ "`wc -l < $(filter %_${TGT}.al, $+)`" == "`wc -l < $(filter %.id, $+)`" ]]


############################################################
# HELPERS
########################################
# Check whether sorting changed sentence order.
.PHONY: check.sorted

check.sorted: $(addprefix check.sorted., ${CORPORA_FILES})

check.sorted.%: %.db.sorted %.db.unsorted
	-diff -q $*.db.unsorted $*.db.sorted || true

%.db.unsorted:  %.${TMX_SRC}.clean %.${TMX_TGT}.clean %.id.tmp %.id
	paste $*.id.tmp $*.${TMX_SRC}.clean $*.${TMX_TGT}.clean \
	| perl -ne 'print $$_ unless /(^|\t)(\t|$$)/;' \
	> $*.db.unsorted

%.db.sorted:  %.db.unsorted
	cat $*.db.unsorted \
	| ${DO_SORT} \
	> $*.db.sorted
	
########################################
# Check the counts.
.PHONY: check.counts

check.counts: $(addprefix check.counts., ${CORPORA_FILES})

check.counts.%:  %.utf8.tmx
	${MAKE} counts.$*
	${MAKE} empty.$*
	@echo

########################################
# Count the number of expected and extracted sentences.
.PHONY: counts
counts: $(addprefix counts., ${CORPORA_FILES})

counts.%: %.utf8.tmx
	-egrep '<tu>|<tu .*>' $< | wc -l
	-wc -l $(addprefix $*, _${SRC}.al _${TGT}.al .id .${TMX_SRC} .${TMX_TGT} .${TMX_SRC}.clean .${TMX_TGT}.clean) || true

########################################
# Count empty lines. The difference should be the missing tuv/seg.
.PHONY: empty
empty: $(addprefix empty., ${CORPORA_FILES})

empty.%: %.utf8.tmx
	-egrep -H -c $$'<seg></seg>' $< || true
	-egrep -c '^ *$$' $(addprefix $*, _${SRC}.al _${TGT}.al .id .${TMX_SRC} .${TMX_TGT}) || true

########################################
# Get counts for all non latin1 characters.
.PHONY: non_latin1_counts
non_latin1_counts:
	cat $(CORPORA_FILES:=_${SRC}.al) $(CORPORA_FILES:=_${TGT}.al) \
	| iconv_libiconv -f UTF-8 -t latin1 --unicode-subst="<[%x]>" \
	| egrep -o '<\[[^<]+\]>' \
	| sort \
	| uniq -c
	
.PHONY: check.non_latin1
check.non_latin1: $(addprefix check.non_latin1., ${CORPORA_FILES})

check.non_latin1.%: check.nl.%_${SRC}.al check.nl.%_${TGT}.al
	@echo 

check.nl.%: %
	cat $< \
	| iconv_libiconv -f UTF-8 -t latin1 --unicode-subst="<[%x]>" \
	| egrep -o '<\[[^<]+\]>' \
	| sort \
	| uniq -c

########################################
# Look for html markup in all .al files.
.PHONY: find_markup
find_markup: find_markup.al

# Look for html markup in all files having % extension.
find_markup.%:
	-egrep '<[^<]+>|\{\\' *.$* || true

########################################
# Find sentences that appear three times or more.
.PHONY: duplicates
duplicates:
	sort $(CORPORA_FILES:=_${TGT}.al) | uniq -c | egrep -v '[ ]*1 ' | egrep -v '[ ]*2 ' | sort -nr

---
title: ROOT 7
---


For the first time since 20 year (i.e. ever), the ROOT team plans to break backward
compatibility for crucial interfaces - once. This new major version of ROOT will make
ROOT much simpler and safer to use: we want to increase clarity and usability. If you
are a physicist, please read on - this is about your ROOT.

The ROOT team will be releasing parts of ROOT 7 throughout the coming years. Previews
will gradually sneak into the ROOT sources, in the `namespace ROOT::Experimental` for
those parts that are not yet cast in stone, and in the namespace ROOT for those that
are. We will use standard C++ types, standard interface behavior (e.g. with respect
to ownership and thread safety), good documentation and tests: we are trying to be nice!

## ROOT 7 Development

We meet about once a month to discuss topics and development of the new ROOT 7 interfaces.
You can subscribe to root7-discussion@cern.ch to get the meeting announcements, and
to participate at the discussions. The meeting presentations are available in Indico
as "ROOT Interface Discussion".

We are happy to receive pull requests. Please submit them to our
[Github repo](https://github.com/root-mirror/root).

## Feedback

The main point of the meeting and this page is to solicit your feedback. Most of
it has been taken care of in the code already.

## Building ROOT 7

### Pre-requisites

Support for the c++14 standard is required. Usage of g++ >= 5 or clang >= 3.4 is recommended.

### Relevant cmake variables

The `cxx14` cmake variables must be "ON".
Building from source would look similar to this:

```
$ mkdir root7_build
$ cd root7_build
$ cmake -Dcxx14=ON path/to/root/source
$ cmake --build . -- -j4
```

### Examples

#### Fill a series of histograms

##### New interfaces

``` cpp
#include "ROOT/TAxis.h"
#include "ROOT/TFile.h"
#include "ROOT/THist.h"
#include "TTreeReader.h"
#include "TTreeReaderArray.h"
#include "TTree.h"

// Another function. Who knows what it does in a month from now.
void someOtherFunction();

void fill(TTree* tree) {
  using namespace ROOT::Experimental;
  // Define the axes, we will re-use them
  TAxisConfig axisMuPt("muon pT [GeV]", {0., 1., 10., 100.});
  TAxisConfig axisJetEt("jet ET [GeV]", 10, 0., 1000.);
  TAxisConfig axisTag("tag value", 10, 0., 1.);

  // Create the histograms
  TH2F hMuPtTag("muon pT versus tag value", axisMuPt, axisTag);
  TH2F hJetEtTag("jet ET versus tag value", axisJetEt, axisTag);

  // Set up reading from the TTree
  TTreeReader reader(tree);
  TTreeReaderArray<float> jetEt(reader, "jet.ET");
  TTreeReaderArray<float> muPt(reader, "jet.lead_mu.pT");
  TTreeReaderArray<float> tag(reader, "jet.tag");

  // Fill the histograms
  while (reader.Next()) {
    for (int iJet = 0; iJet < jetEt.size(); ++iJet) {
      hMuPtTag.Fill({muPt[iJet], tag[iJet]});
      hJetEtTag.Fill({jetEt[iJet], tag[iJet]});
    }
  }

  someOtherFunction();

  // Store the result; can create the file where really needed.
  TFilePtr file = TFile::Recreate("jetmuontag.root");
  file->Write("hist_mupt", hMuPtTag);
  file->Write("hist_jetet", hJetEtTag);
}
```

##### ROOT6 interfaces

```.cpp
#include "TFile.h"
#include "TH2.h"
#include "TTreeReader.h"
#include "TTreeReaderArray.h"
#include "TTree.h"

// Another function. Who knows what it does in a month from now.
void someOtherFunction();

void fill(TTree* tree) {
  // Create the file before so it can own the histograms.
  TFile* file = TFile::Open("jetmuontag.root", "RECREATE");

  // Create the histograms (cannot mix fixed and variable size bins)
  const double muonPtBins[] = {0., 1., 10., 100.};
  TH2* hMuPtTag = new TH2F("hMuPtTag", "muon pT versus tag value;muon pT [GeV];tag value",
    sizeof(muonPtBins) - 1, muonPtBins, 10, 0., 1.);
  TH2* hJetEtTag = new TH2F("hJetEtTag", "jet ET versus tag value;jet ET [GeV];tag value",
    10, 0., 1000., 10, 0., 1.);

  // Set up reading from the TTree
  TTreeReader reader(tree);
  TTreeReaderArray<float> jetEt(reader, "jet.ET");
  TTreeReaderArray<float> muPt(reader, "jet.lead_mu.pT");
  TTreeReaderArray<float> tag(reader, "jet.tag");

  // Fill the histograms
  while (reader.Next()) {
    for (int iJet = 0; iJet < jetEt.size(); ++iJet) {
      hMuPtTag->Fill(muPt[iJet], tag[iJet]);
      hJetEtTag->Fill(jetEt[iJet], tag[iJet]);
    }
  }

  someOtherFunction();

  // Store the result. Ideally using file->Write() but very few people do that.
  hMuPtTag->Write();
  hJetEtTag->Write();

  delete file; // but not the histograms!
}
```

But the new interfaces are not about shortening your code - but about robustness.
Here are a few examples of what can go wrong with the ROOT6 interfaces:

##### ROOT6 interfaces, showing what can go wrong.

```.cpp
#include "TFile.h"
#include "TH2.h"
#include "TTreeReader.h"
#include "TTreeReaderArray.h"
#include "TTree.h"

// Another function. Who knows what it does in a month from now.
void someOtherFunction();

void fill(TTree* tree) {
  // Create the file before so it can own the histograms.
  TFile* file = TFile::Open("jetmuontag.root", "RECRAETE");

  // Create the histograms (cannot mix fixed and variable size bins)
  const double muonPtBins[] = {0., 1., 10., 100.};
  // The axis titles might have been changed. Impossible to see.
  TH2* hMuPtTag = new TH2F("hMuPtTag", "muon pT versus tag value;tag value;muon pT [GeV]",
    4, muonPtBins,  10, 0., 1.);
  TH1* hJetEt = new TH1F("hJetEt", "jet ET versus tag value;jet ET [GeV];tag value",
    10, 0., 1000.);

  // Set up reading from the TTree
  TTreeReader reader(tree);
  TTreeReaderArray<float> jetEt(reader, "jet.ET");
  TTreeReaderArray<float> muPt(reader, "jet.lead_mu.pT");
  TTreeReaderArray<float> tag(reader, "jet.tag");

  // Fill the histograms
  while (reader.Next()) {
    for (int iJet = 0; iJet < jetEt.size(); ++iJet) {
      hMuPtTag->Fill(muPt[iJet], tag[iJet]);
      hJetEtTag->Fill(jetEt[iJet], tag[iJet]);
    }
  }

  someOtherFunction();

  // Store the result. Ideally using file->Write() but very few people do that.
  hMuPtTag->Write();
  hJetEtTag->Write();

  delete file; // but not the histograms!
}
```

Here are the problems:

 - Constructing the `TFile`, `"RECRAETE"` is misspelled.
 - The axis titles of `hMuPtJetET` are inverted.
 - The number of `muonPtBins` is wrong.
 - The histogram `hJetEtTag` is filled with weights of `*tag;` that might be a
   leftover y coordinate.
 - You get two copies of the histogram in the file, one for `histogram->Write()`,
   one because the histogram is already associated with the file and will be
   written on file destruction.
 - The call to `someOtherFunction()` might change `gDirectory` and thus `hMuPtJetET`
   might not be written to `jetmuontag.root`.

To track these problems down you'd have to spend your time debugging them.
Instead, the new interfaces will simply not allow this to happen: no debugging needed!
# barcode-zxing-itf14

package com.google.zxing.oned;

import com.google.zxing.BarcodeFormat;
import com.google.zxing.ReaderException;
import com.google.zxing.Result;
import com.google.zxing.ResultPoint;
import com.google.zxing.common.BitArray;
import com.google.zxing.common.GenericResultPoint;
import java.util.Hashtable;
public final class ITF14Reader extends AbstractOneDReader {
  private static final int MAX_AVG_VARIANCE = (int) (PATTERN_MATCH_RESULT_SCALE_FACTOR * 0.42f);
  private static final int MAX_INDIVIDUAL_VARIANCE = (int) (PATTERN_MATCH_RESULT_SCALE_FACTOR * 0.7f);
  private static final int W = 3; // Pixel width of a wide line
  private static final int N = 1; // Pixed width of a narrow line
  private final int DIGIT_COUNT = 14;  // There are 14 digits in ITF-14
  private static final int[] START_PATTERN = {N, N, N, N};
  private static final int[] END_PATTERN_REVERSED = {N, N, W};
  static final int[][] PATTERNS = {
      {N, N, W, W, N}, // 0
      {W, N, N, N, W}, // 1
      {N, W, N, N, W}, // 2
      {W, W, N, N, N}, // 3
      {N, N, W, N, W}, // 4
      {W, N, W, N, N}, // 5
      {N, W, W, N, N}, // 6
      {N, N, N, W, W}, // 7
      {W, N, N, W, N}, // 8
      {N, W, N, W, N}  // 9
  };

  public final Result decodeRow(int rowNumber, BitArray row, Hashtable hints) throws ReaderException {

    StringBuffer result = new StringBuffer(20);
    int[] startRange = decodeStart(row);
    int[] endRange = decodeEnd(row);
    decodeMiddle(row, startRange[1], endRange[0], result);
    String resultString = result.toString();
    if (!AbstractUPCEANReader.checkStandardUPCEANChecksum(resultString)) {
      throw ReaderException.getInstance();
    }
    return new Result(resultString, null, // no natural byte representation
        // for these barcodes
        new ResultPoint[]{new GenericResultPoint(startRange[1], (float) rowNumber),
            new GenericResultPoint(startRange[0], (float) rowNumber)},
        BarcodeFormat.ITF_14);
  }

  protected void decodeMiddle(BitArray row, int payloadStart, int payloadEnd, StringBuffer resultString)
      throws ReaderException {
    int[] counterDigitPair = new int[10];
    int[] counterBlack = new int[5];
    int[] counterWhite = new int[5];
    for (int x = 0; x < DIGIT_COUNT / 2 && payloadStart < payloadEnd; x++) {

      recordPattern(row, payloadStart, counterDigitPair);
       for (int k = 0; k < 5; k++) {
        counterBlack[k] = counterDigitPair[k * 2];
        counterWhite[k] = counterDigitPair[(k * 2) + 1];
      }
      int bestMatch = decodeDigit(counterBlack);
      resultString.append((char) ('0' + bestMatch % 10));
      bestMatch = decodeDigit(counterWhite);
      resultString.append((char) ('0' + bestMatch % 10));
      for (int i = 0; i < counterDigitPair.length; i++) {
        payloadStart += counterDigitPair[i];
      }
    }
  }

  int[] decodeStart(BitArray row) throws ReaderException {
    int endStart = skipWhiteSpace(row);
    return findGuardPattern(row, endStart, START_PATTERN);
  }

 private int skipWhiteSpace(BitArray row) throws ReaderException {
    int width = row.getSize();
    int endStart = 0;
    while (endStart < width) {
      if (row.get(endStart)) {
        break;
      }
      endStart++;
    }
    if (endStart == width)
      throw ReaderException.getInstance();

    return endStart;
  }

 int[] decodeEnd(BitArray row) throws ReaderException {
    row.reverse();
    int endStart = skipWhiteSpace(row);
    int end[];
    try {
      end = findGuardPattern(row, endStart, END_PATTERN_REVERSED);
    } catch (ReaderException e) {
      row.reverse();
      throw e;
    }
    int temp = end[0];
    end[0] = row.getSize() - end[1];
    end[1] = row.getSize() - temp;
    row.reverse();
    return end;
  }

  int[] findGuardPattern(BitArray row, int rowOffset, int[] pattern) throws ReaderException {
    int patternLength = pattern.length;
    int[] counters = new int[patternLength];
    int width = row.getSize();
    boolean isWhite = false;

    int counterPosition = 0;
    int patternStart = rowOffset;
    for (int x = rowOffset; x < width; x++) {
      boolean pixel = row.get(x);
      if ((!pixel && isWhite) || (pixel && !isWhite)) {
        counters[counterPosition]++;
      } else {
        if (counterPosition == patternLength - 1) {
          if (patternMatchVariance(counters, pattern, MAX_INDIVIDUAL_VARIANCE) < MAX_AVG_VARIANCE) {
            return new int[]{patternStart, x};
          }
          patternStart += counters[0] + counters[1];
          for (int y = 2; y < patternLength; y++) {
            counters[y - 2] = counters[y];
          }
          counters[patternLength - 2] = 0;
          counters[patternLength - 1] = 0;
          counterPosition--;
        } else {
          counterPosition++;
        }
        counters[counterPosition] = 1;
        isWhite = !isWhite;
      }
    }
    throw ReaderException.getInstance();
  }

   static int decodeDigit(int[] counters) throws ReaderException {

    int bestVariance = MAX_AVG_VARIANCE; // worst variance we'll accept
    int bestMatch = -1;
    int max = PATTERNS.length;
    for (int i = 0; i < max; i++) {
      int[] pattern = PATTERNS[i];
      int variance = patternMatchVariance(counters, pattern, MAX_INDIVIDUAL_VARIANCE);
      if (variance < bestVariance) {
        bestVariance = variance;
        bestMatch = i;
      }
    }
    if (bestMatch >= 0) {
      return bestMatch;
    } else {
      throw ReaderException.getInstance();
		}
	}

}

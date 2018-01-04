# Scan Multiple ADC Channels

The following are the steps to scan a number of STM32's ADC channels (polling technique).

1. Configure the ADC channel pins as analog.
2. Configure ADC to scan mode (`CR1:SCAN` = 1), data being right-aligned (for ease of reading), generate `EOC` (end-of-conversion) at the end of each channel conversion (`CR2:EOCS` = 1), and turn on ADC (`CR2:ADON`).
3. Register the channels to be scanned in the Sequence Register(s).
4. Clear all the flags (if some error flags are asserted, the conversion will not start)
5. Start the conversion (`CR2:SWSTART`)
6. Check the `EOC` (end-of-conversion) flag. If it is set, proceed to step 7. Otherwise repeat step 6.
7. Clear `EOC` flag
8. Read the converted data (`DR`)
9. Repeat step 6, if the is anymore channel left to scan. Otherwise stop.

**Important note:** to permit successful polling, `EOCS` must be configured to _generate `EOC` on each channel conversion_. Otherwise, an overrun error will be triggered. This happens because there is only one DR register to read the converted value of all channels. Since each channel is converted one-by-one, we need to know when each conversion has ended. Also the error flags must be cleared before a new software-triggered conversion command is issued (through `SWSTART` bit).  

The following is a C program example (though incomplete code). It scans ADC1 channels 5 and 13 and print the raw result.

```C
  int channelSeq[] = {5, 13};

  // Un-reset ADC1 and clock it.
  enableAdc(ADC1_DEV);

  // Set the sampling time for channels 5 and 13
  adcChannelSamplingTime(adc1, 5, ADC_3_CYCLES);
  adcChannelSamplingTime(adc1, 13, ADC_480_CYCLES);
  // Register the channels in the sequence register(s)
  adcSetChannelSamplingSequence(adc1, channelSeq, sizeof(channelSeq)/4);
  // Configure the ADC1
  adc1->CR1 = ADC_12_BIT_RES | ADC_SCAN;
  adc1->CR2 = ADC_RIGHT_ALIGN | ADC_SINGLE_END_OF_CONV | ADC_ON;

  // Configure the ADC channel pins as analog
  gpioConfig(GpioA, 5, GPIO_MODE_ANA, GPIO_NO_DRIVE, GPIO_NO_PULL,      \
             GPIO_NO_SPEED);
  gpioConfig(GpioC, 3, GPIO_MODE_ANA, GPIO_NO_DRIVE, GPIO_NO_PULL,      \
             GPIO_NO_SPEED);

  // Delay for at least 1usec after ADON flag is asserted (turn on ADC1).
  HAL_Delay(1);

  // Clear all flags
  adcClearAllFlags(adc1);
  while(1) { 
    // Trigger the conversion by software   
    adcStartConversion(adc1);
    // Wait till the first end-of-conversion happens
    while(!adcHasConversionEnded(adc1));
    adcClearFlags(adc1, ADC_EOC);
    // Read the converted data
    chn5 = adcReadValue(adc1);
    // Wait till the second end-of-conversion happens    
    while(!adcHasConversionEnded(adc1));
    adcClearFlags(adc1, ADC_EOC);
    // Read the next converted data
    chn13 = adcReadValue(adc1);
    // Print the result    
    printf("channel5:%d, channel13:%d\n", chn5, chn13);
  }

```

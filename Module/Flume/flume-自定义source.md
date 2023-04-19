来源： https://blog.csdn.net/xiao_jun_0820/article/details/38312091

我想实现一个功能，就在读一个文件的时候，将文件的名字和文件生成的日期作为event的header传到hdfs上时，不同的event存到不同的目录下，如一个文件是a.log.2014-07-25在hdfs上是存到/a/2014-07-25目录下，a.log.2014-07-26存到/a/2014-07-26目录下，就是每个文件对应自己的目录。

带着这个问题，我又重新翻看了官方的文档，发现一个spooling directory source跟这个需求稍微有点吻合：它监视指定的文件夹下面有没有写入新的文件，有的话，就会把该文件内容传递给sink,然后将该文件后缀标示为.complete，表示已处理。提供了参数可以将文件名和文件全路径名添加到event的header中去。

现有的功能不能满足我们的需求，但是至少提供了一个方向：它能将文件名放入header！这样我们在这个地方稍微修改修改，把文件名拆分一下，然后再放入header，这样就完成了我们想要的功能了。

于是就打开了源代码，果然不复杂，代码结构非常清晰，按照我的思路，稍微改了一下，就实现了这个功能，主要修改了与spooling directory source代码相关的三个类，分别是：ReliableSpoolingFileEventExtReader，SpoolDirectorySourceConfigurationExtConstants，SpoolDirectoryExtSource（在原类名的基础上增加了：Ext）代码如下：


```java
/*
 * Licensed to the Apache Software Foundation (ASF) under one
 * or more contributor license agreements.  See the NOTICE file
 * distributed with this work for additional information
 * regarding copyright ownership.  The ASF licenses this file
 * to you under the Apache License, Version 2.0 (the
 * "License"); you may not use this file except in compliance
 * with the License.  You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */

package com.besttone.flume;

import java.io.File;
import java.io.FileFilter;
import java.io.FileNotFoundException;
import java.io.IOException;
import java.nio.charset.Charset;
import java.util.Arrays;
import java.util.Collections;
import java.util.Comparator;
import java.util.List;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

import org.apache.flume.Context;
import org.apache.flume.Event;
import org.apache.flume.FlumeException;
import org.apache.flume.annotations.InterfaceAudience;
import org.apache.flume.annotations.InterfaceStability;
import org.apache.flume.client.avro.ReliableEventReader;
import org.apache.flume.serialization.DecodeErrorPolicy;
import org.apache.flume.serialization.DurablePositionTracker;
import org.apache.flume.serialization.EventDeserializer;
import org.apache.flume.serialization.EventDeserializerFactory;
import org.apache.flume.serialization.PositionTracker;
import org.apache.flume.serialization.ResettableFileInputStream;
import org.apache.flume.serialization.ResettableInputStream;
import org.apache.flume.tools.PlatformDetect;
import org.joda.time.DateTime;
import org.joda.time.format.DateTimeFormat;
import org.joda.time.format.DateTimeFormatter;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.google.common.base.Charsets;
import com.google.common.base.Optional;
import com.google.common.base.Preconditions;
import com.google.common.io.Files;

/**
 * <p/>
 * A {@link ReliableEventReader} which reads log data from files stored in a
 * spooling directory and renames each file once all of its data has been read
 * (through {@link EventDeserializer#readEvent()} calls). The user must
 * {@link #commit()} each read, to indicate that the lines have been fully
 * processed.
 * <p/>
 * Read calls will return no data if there are no files left to read. This
 * class, in general, is not thread safe.
 * 
 * <p/>
 * This reader assumes that files with unique file names are left in the
 * spooling directory and not modified once they are placed there. Any user
 * behavior which violates these assumptions, when detected, will result in a
 * FlumeException being thrown.
 * 
 * <p/>
 * This class makes the following guarantees, if above assumptions are met:
 * <ul>
 * <li>Once a log file has been renamed with the {@link #completedSuffix}, all
 * of its records have been read through the
 * {@link EventDeserializer#readEvent()} function and {@link #commit()}ed at
 * least once.
 * <li>All files in the spooling directory will eventually be opened and
 * delivered to a {@link #readEvents(int)} caller.
 * </ul>
 */
@InterfaceAudience.Private
@InterfaceStability.Evolving
public class ReliableSpoolingFileEventExtReader implements ReliableEventReader {

	private static final Logger logger = LoggerFactory
			.getLogger(ReliableSpoolingFileEventExtReader.class);

	static final String metaFileName = ".flumespool-main.meta";

	private final File spoolDirectory;
	private final String completedSuffix;
	private final String deserializerType;
	private final Context deserializerContext;
	private final Pattern ignorePattern;
	private final File metaFile;
	private final boolean annotateFileName;
	private final boolean annotateBaseName;
	private final String fileNameHeader;
	private final String baseNameHeader;

	// 添加参数开始
	private final boolean annotateFileNameExtractor;
	private final String fileNameExtractorHeader;
	private final Pattern fileNameExtractorPattern;
	private final boolean convertToTimestamp;
	private final String dateTimeFormat;

	private final boolean splitFileName;
	private final String splitBy;
	private final String splitBaseNameHeader;
	// 添加参数结束

	private final String deletePolicy;
	private final Charset inputCharset;
	private final DecodeErrorPolicy decodeErrorPolicy;

	private Optional<FileInfo> currentFile = Optional.absent();
	/** Always contains the last file from which lines have been read. **/
	private Optional<FileInfo> lastFileRead = Optional.absent();
	private boolean committed = true;

	/**
	 * Create a ReliableSpoolingFileEventReader to watch the given directory.
	 */
	private ReliableSpoolingFileEventExtReader(File spoolDirectory,
			String completedSuffix, String ignorePattern,
			String trackerDirPath, boolean annotateFileName,
			String fileNameHeader, boolean annotateBaseName,
			String baseNameHeader, String deserializerType,
			Context deserializerContext, String deletePolicy,
			String inputCharset, DecodeErrorPolicy decodeErrorPolicy,
			boolean annotateFileNameExtractor, String fileNameExtractorHeader,
			String fileNameExtractorPattern, boolean convertToTimestamp,
			String dateTimeFormat, boolean splitFileName, String splitBy,
			String splitBaseNameHeader) throws IOException {

		// Sanity checks
		Preconditions.checkNotNull(spoolDirectory);
		Preconditions.checkNotNull(completedSuffix);
		Preconditions.checkNotNull(ignorePattern);
		Preconditions.checkNotNull(trackerDirPath);
		Preconditions.checkNotNull(deserializerType);
		Preconditions.checkNotNull(deserializerContext);
		Preconditions.checkNotNull(deletePolicy);
		Preconditions.checkNotNull(inputCharset);

		// validate delete policy
		if (!deletePolicy.equalsIgnoreCase(DeletePolicy.NEVER.name())
				&& !deletePolicy
						.equalsIgnoreCase(DeletePolicy.IMMEDIATE.name())) {
			throw new IllegalArgumentException("Delete policies other than "
					+ "NEVER and IMMEDIATE are not yet supported");
		}

		if (logger.isDebugEnabled()) {
			logger.debug("Initializing {} with directory={}, metaDir={}, "
					+ "deserializer={}", new Object[] {
					ReliableSpoolingFileEventExtReader.class.getSimpleName(),
					spoolDirectory, trackerDirPath, deserializerType });
		}

		// Verify directory exists and is readable/writable
		Preconditions
				.checkState(
						spoolDirectory.exists(),
						"Directory does not exist: "
								+ spoolDirectory.getAbsolutePath());
		Preconditions.checkState(spoolDirectory.isDirectory(),
				"Path is not a directory: " + spoolDirectory.getAbsolutePath());

		// Do a canary test to make sure we have access to spooling directory
		try {
			File canary = File.createTempFile("flume-spooldir-perm-check-",
					".canary", spoolDirectory);
			Files.write("testing flume file permissions\n", canary,
					Charsets.UTF_8);
			List<String> lines = Files.readLines(canary, Charsets.UTF_8);
			Preconditions.checkState(!lines.isEmpty(), "Empty canary file %s",
					canary);
			if (!canary.delete()) {
				throw new IOException("Unable to delete canary file " + canary);
			}
			logger.debug("Successfully created and deleted canary file: {}",
					canary);
		} catch (IOException e) {
			throw new FlumeException("Unable to read and modify files"
					+ " in the spooling directory: " + spoolDirectory, e);
		}

		this.spoolDirectory = spoolDirectory;
		this.completedSuffix = completedSuffix;
		this.deserializerType = deserializerType;
		this.deserializerContext = deserializerContext;
		this.annotateFileName = annotateFileName;
		this.fileNameHeader = fileNameHeader;
		this.annotateBaseName = annotateBaseName;
		this.baseNameHeader = baseNameHeader;
		this.ignorePattern = Pattern.compile(ignorePattern);
		this.deletePolicy = deletePolicy;
		this.inputCharset = Charset.forName(inputCharset);
		this.decodeErrorPolicy = Preconditions.checkNotNull(decodeErrorPolicy);

		// 增加代码开始
		this.annotateFileNameExtractor = annotateFileNameExtractor;
		this.fileNameExtractorHeader = fileNameExtractorHeader;
		this.fileNameExtractorPattern = Pattern
				.compile(fileNameExtractorPattern);
		this.convertToTimestamp = convertToTimestamp;
		this.dateTimeFormat = dateTimeFormat;

		this.splitFileName = splitFileName;
		this.splitBy = splitBy;
		this.splitBaseNameHeader = splitBaseNameHeader;
		// 增加代码结束

		File trackerDirectory = new File(trackerDirPath);

		// if relative path, treat as relative to spool directory
		if (!trackerDirectory.isAbsolute()) {
			trackerDirectory = new File(spoolDirectory, trackerDirPath);
		}

		// ensure that meta directory exists
		if (!trackerDirectory.exists()) {
			if (!trackerDirectory.mkdir()) {
				throw new IOException(
						"Unable to mkdir nonexistent meta directory "
								+ trackerDirectory);
			}
		}

		// ensure that the meta directory is a directory
		if (!trackerDirectory.isDirectory()) {
			throw new IOException("Specified meta directory is not a directory"
					+ trackerDirectory);
		}

		this.metaFile = new File(trackerDirectory, metaFileName);
	}

	/**
	 * Return the filename which generated the data from the last successful
	 * {@link #readEvents(int)} call. Returns null if called before any file
	 * contents are read.
	 */
	public String getLastFileRead() {
		if (!lastFileRead.isPresent()) {
			return null;
		}
		return lastFileRead.get().getFile().getAbsolutePath();
	}

	// public interface
	public Event readEvent() throws IOException {
		List<Event> events = readEvents(1);
		if (!events.isEmpty()) {
			return events.get(0);
		} else {
			return null;
		}
	}

	public List<Event> readEvents(int numEvents) throws IOException {
		if (!committed) {
			if (!currentFile.isPresent()) {
				throw new IllegalStateException("File should not roll when "
						+ "commit is outstanding.");
			}
			logger.info("Last read was never committed - resetting mark position.");
			currentFile.get().getDeserializer().reset();
		} else {
			// Check if new files have arrived since last call
			if (!currentFile.isPresent()) {
				currentFile = getNextFile();
			}
			// Return empty list if no new files
			if (!currentFile.isPresent()) {
				return Collections.emptyList();
			}
		}

		EventDeserializer des = currentFile.get().getDeserializer();
		List<Event> events = des.readEvents(numEvents);

		/*
		 * It's possible that the last read took us just up to a file boundary.
		 * If so, try to roll to the next file, if there is one.
		 */
		if (events.isEmpty()) {
			retireCurrentFile();
			currentFile = getNextFile();
			if (!currentFile.isPresent()) {
				return Collections.emptyList();
			}
			events = currentFile.get().getDeserializer().readEvents(numEvents);
		}

		if (annotateFileName) {
			String filename = currentFile.get().getFile().getAbsolutePath();
			for (Event event : events) {
				event.getHeaders().put(fileNameHeader, filename);
			}
		}

		if (annotateBaseName) {
			String basename = currentFile.get().getFile().getName();
			for (Event event : events) {
				event.getHeaders().put(baseNameHeader, basename);
			}
		}

		// 增加代码开始

		// 按正则抽取文件名的内容
		if (annotateFileNameExtractor) {

			Matcher matcher = fileNameExtractorPattern.matcher(currentFile
					.get().getFile().getName());

			if (matcher.find()) {
				String value = matcher.group();
				if (convertToTimestamp) {
					DateTimeFormatter formatter = DateTimeFormat
							.forPattern(dateTimeFormat);
					DateTime dateTime = formatter.parseDateTime(value);

					value = Long.toString(dateTime.getMillis());
				}

				for (Event event : events) {
					event.getHeaders().put(fileNameExtractorHeader, value);
				}
			}

		}

		// 按分隔符拆分文件名
		if (splitFileName) {
			String[] splits = currentFile.get().getFile().getName()
					.split(splitBy);

			for (Event event : events) {
				for (int i = 0; i < splits.length; i++) {
					event.getHeaders().put(splitBaseNameHeader + i, splits[i]);
				}

			}

		}

		// 增加代码结束

		committed = false;
		lastFileRead = currentFile;
		return events;
	}

	@Override
	public void close() throws IOException {
		if (currentFile.isPresent()) {
			currentFile.get().getDeserializer().close();
			currentFile = Optional.absent();
		}
	}

	/** Commit the last lines which were read. */
	@Override
	public void commit() throws IOException {
		if (!committed && currentFile.isPresent()) {
			currentFile.get().getDeserializer().mark();
			committed = true;
		}
	}

	/**
	 * Closes currentFile and attempt to rename it.
	 * 
	 * If these operations fail in a way that may cause duplicate log entries,
	 * an error is logged but no exceptions are thrown. If these operations fail
	 * in a way that indicates potential misuse of the spooling directory, a
	 * FlumeException will be thrown.
	 * 
	 * @throws FlumeException
	 *             if files do not conform to spooling assumptions
	 */
	private void retireCurrentFile() throws IOException {
		Preconditions.checkState(currentFile.isPresent());

		File fileToRoll = new File(currentFile.get().getFile()
				.getAbsolutePath());

		currentFile.get().getDeserializer().close();

		// Verify that spooling assumptions hold
		if (fileToRoll.lastModified() != currentFile.get().getLastModified()) {
			String message = "File has been modified since being read: "
					+ fileToRoll;
			throw new IllegalStateException(message);
		}
		if (fileToRoll.length() != currentFile.get().getLength()) {
			String message = "File has changed size since being read: "
					+ fileToRoll;
			throw new IllegalStateException(message);
		}

		if (deletePolicy.equalsIgnoreCase(DeletePolicy.NEVER.name())) {
			rollCurrentFile(fileToRoll);
		} else if (deletePolicy.equalsIgnoreCase(DeletePolicy.IMMEDIATE.name())) {
			deleteCurrentFile(fileToRoll);
		} else {
			// TODO: implement delay in the future
			throw new IllegalArgumentException("Unsupported delete policy: "
					+ deletePolicy);
		}
	}

	/**
	 * Rename the given spooled file
	 * 
	 * @param fileToRoll
	 * @throws IOException
	 */
	private void rollCurrentFile(File fileToRoll) throws IOException {

		File dest = new File(fileToRoll.getPath() + completedSuffix);
		logger.info("Preparing to move file {} to {}", fileToRoll, dest);

		// Before renaming, check whether destination file name exists
		if (dest.exists() && PlatformDetect.isWindows()) {
			/*
			 * If we are here, it means the completed file already exists. In
			 * almost every case this means the user is violating an assumption
			 * of Flume (that log files are placed in the spooling directory
			 * with unique names). However, there is a corner case on Windows
			 * systems where the file was already rolled but the rename was not
			 * atomic. If that seems likely, we let it pass with only a warning.
			 */
			if (Files.equal(currentFile.get().getFile(), dest)) {
				logger.warn("Completed file " + dest
						+ " already exists, but files match, so continuing.");
				boolean deleted = fileToRoll.delete();
				if (!deleted) {
					logger.error("Unable to delete file "
							+ fileToRoll.getAbsolutePath()
							+ ". It will likely be ingested another time.");
				}
			} else {
				String message = "File name has been re-used with different"
						+ " files. Spooling assumptions violated for " + dest;
				throw new IllegalStateException(message);
			}

			// Dest file exists and not on windows
		} else if (dest.exists()) {
			String message = "File name has been re-used with different"
					+ " files. Spooling assumptions violated for " + dest;
			throw new IllegalStateException(message);

			// Destination file does not already exist. We are good to go!
		} else {
			boolean renamed = fileToRoll.renameTo(dest);
			if (renamed) {
				logger.debug("Successfully rolled file {} to {}", fileToRoll,
						dest);

				// now we no longer need the meta file
				deleteMetaFile();
			} else {
				/*
				 * If we are here then the file cannot be renamed for a reason
				 * other than that the destination file exists (actually, that
				 * remains possible w/ small probability due to TOC-TOU
				 * conditions).
				 */
				String message = "Unable to move "
						+ fileToRoll
						+ " to "
						+ dest
						+ ". This will likely cause duplicate events. Please verify that "
						+ "flume has sufficient permissions to perform these operations.";
				throw new FlumeException(message);
			}
		}
	}

	/**
	 * Delete the given spooled file
	 * 
	 * @param fileToDelete
	 * @throws IOException
	 */
	private void deleteCurrentFile(File fileToDelete) throws IOException {
		logger.info("Preparing to delete file {}", fileToDelete);
		if (!fileToDelete.exists()) {
			logger.warn("Unable to delete nonexistent file: {}", fileToDelete);
			return;
		}
		if (!fileToDelete.delete()) {
			throw new IOException("Unable to delete spool file: "
					+ fileToDelete);
		}
		// now we no longer need the meta file
		deleteMetaFile();
	}

	/**
	 * Find and open the oldest file in the chosen directory. If two or more
	 * files are equally old, the file name with lower lexicographical value is
	 * returned. If the directory is empty, this will return an absent option.
	 */
	private Optional<FileInfo> getNextFile() {
		/* Filter to exclude finished or hidden files */
		FileFilter filter = new FileFilter() {
			public boolean accept(File candidate) {
				String fileName = candidate.getName();
				if ((candidate.isDirectory())
						|| (fileName.endsWith(completedSuffix))
						|| (fileName.startsWith("."))
						|| ignorePattern.matcher(fileName).matches()) {
					return false;
				}
				return true;
			}
		};
		List<File> candidateFiles = Arrays.asList(spoolDirectory
				.listFiles(filter));
		if (candidateFiles.isEmpty()) {
			return Optional.absent();
		} else {
			Collections.sort(candidateFiles, new Comparator<File>() {
				public int compare(File a, File b) {
					int timeComparison = new Long(a.lastModified())
							.compareTo(new Long(b.lastModified()));
					if (timeComparison != 0) {
						return timeComparison;
					} else {
						return a.getName().compareTo(b.getName());
					}
				}
			});
			File nextFile = candidateFiles.get(0);
			try {
				// roll the meta file, if needed
				String nextPath = nextFile.getPath();
				PositionTracker tracker = DurablePositionTracker.getInstance(
						metaFile, nextPath);
				if (!tracker.getTarget().equals(nextPath)) {
					tracker.close();
					deleteMetaFile();
					tracker = DurablePositionTracker.getInstance(metaFile,
							nextPath);
				}

				// sanity check
				Preconditions
						.checkState(
								tracker.getTarget().equals(nextPath),
								"Tracker target %s does not equal expected filename %s",
								tracker.getTarget(), nextPath);

				ResettableInputStream in = new ResettableFileInputStream(
						nextFile, tracker,
						ResettableFileInputStream.DEFAULT_BUF_SIZE,
						inputCharset, decodeErrorPolicy);
				EventDeserializer deserializer = EventDeserializerFactory
						.getInstance(deserializerType, deserializerContext, in);

				return Optional.of(new FileInfo(nextFile, deserializer));
			} catch (FileNotFoundException e) {
				// File could have been deleted in the interim
				logger.warn("Could not find file: " + nextFile, e);
				return Optional.absent();
			} catch (IOException e) {
				logger.error("Exception opening file: " + nextFile, e);
				return Optional.absent();
			}
		}
	}

	private void deleteMetaFile() throws IOException {
		if (metaFile.exists() && !metaFile.delete()) {
			throw new IOException("Unable to delete old meta file " + metaFile);
		}
	}

	/** An immutable class with information about a file being processed. */
	private static class FileInfo {
		private final File file;
		private final long length;
		private final long lastModified;
		private final EventDeserializer deserializer;

		public FileInfo(File file, EventDeserializer deserializer) {
			this.file = file;
			this.length = file.length();
			this.lastModified = file.lastModified();
			this.deserializer = deserializer;
		}

		public long getLength() {
			return length;
		}

		public long getLastModified() {
			return lastModified;
		}

		public EventDeserializer getDeserializer() {
			return deserializer;
		}

		public File getFile() {
			return file;
		}
	}

	@InterfaceAudience.Private
	@InterfaceStability.Unstable
	static enum DeletePolicy {
		NEVER, IMMEDIATE, DELAY
	}

	/**
	 * Special builder class for ReliableSpoolingFileEventReader
	 */
	public static class Builder {
		private File spoolDirectory;
		private String completedSuffix = SpoolDirectorySourceConfigurationExtConstants.SPOOLED_FILE_SUFFIX;
		private String ignorePattern = SpoolDirectorySourceConfigurationExtConstants.DEFAULT_IGNORE_PAT;
		private String trackerDirPath = SpoolDirectorySourceConfigurationExtConstants.DEFAULT_TRACKER_DIR;
		private Boolean annotateFileName = SpoolDirectorySourceConfigurationExtConstants.DEFAULT_FILE_HEADER;
		private String fileNameHeader = SpoolDirectorySourceConfigurationExtConstants.DEFAULT_FILENAME_HEADER_KEY;
		private Boolean annotateBaseName = SpoolDirectorySourceConfigurationExtConstants.DEFAULT_BASENAME_HEADER;
		private String baseNameHeader = SpoolDirectorySourceConfigurationExtConstants.DEFAULT_BASENAME_HEADER_KEY;
		private String deserializerType = SpoolDirectorySourceConfigurationExtConstants.DEFAULT_DESERIALIZER;
		private Context deserializerContext = new Context();
		private String deletePolicy = SpoolDirectorySourceConfigurationExtConstants.DEFAULT_DELETE_POLICY;
		private String inputCharset = SpoolDirectorySourceConfigurationExtConstants.DEFAULT_INPUT_CHARSET;
		private DecodeErrorPolicy decodeErrorPolicy = DecodeErrorPolicy
				.valueOf(SpoolDirectorySourceConfigurationExtConstants.DEFAULT_DECODE_ERROR_POLICY
						.toUpperCase());

		// 增加代码开始

		private Boolean annotateFileNameExtractor = SpoolDirectorySourceConfigurationExtConstants.DEFAULT_FILENAME_EXTRACTOR;
		private String fileNameExtractorHeader = SpoolDirectorySourceConfigurationExtConstants.DEFAULT_FILENAME_EXTRACTOR_HEADER_KEY;
		private String fileNameExtractorPattern = SpoolDirectorySourceConfigurationExtConstants.DEFAULT_FILENAME_EXTRACTOR_PATTERN;
		private Boolean convertToTimestamp = SpoolDirectorySourceConfigurationExtConstants.DEFAULT_FILENAME_EXTRACTOR_CONVERT_TO_TIMESTAMP;

		private String dateTimeFormat = SpoolDirectorySourceConfigurationExtConstants.DEFAULT_FILENAME_EXTRACTOR_DATETIME_FORMAT;

		private Boolean splitFileName = SpoolDirectorySourceConfigurationExtConstants.DEFAULT_SPLIT_FILENAME;
		private String splitBy = SpoolDirectorySourceConfigurationExtConstants.DEFAULT_SPLITY_BY;
		private String splitBaseNameHeader = SpoolDirectorySourceConfigurationExtConstants.DEFAULT_SPLIT_BASENAME_HEADER;

		public Builder annotateFileNameExtractor(
				Boolean annotateFileNameExtractor) {
			this.annotateFileNameExtractor = annotateFileNameExtractor;
			return this;
		}

		public Builder fileNameExtractorHeader(String fileNameExtractorHeader) {
			this.fileNameExtractorHeader = fileNameExtractorHeader;
			return this;
		}

		public Builder fileNameExtractorPattern(String fileNameExtractorPattern) {
			this.fileNameExtractorPattern = fileNameExtractorPattern;
			return this;
		}

		public Builder convertToTimestamp(Boolean convertToTimestamp) {
			this.convertToTimestamp = convertToTimestamp;
			return this;
		}

		public Builder dateTimeFormat(String dateTimeFormat) {
			this.dateTimeFormat = dateTimeFormat;
			return this;
		}

		public Builder splitFileName(Boolean splitFileName) {
			this.splitFileName = splitFileName;
			return this;
		}

		public Builder splitBy(String splitBy) {
			this.splitBy = splitBy;
			return this;
		}

		public Builder splitBaseNameHeader(String splitBaseNameHeader) {
			this.splitBaseNameHeader = splitBaseNameHeader;
			return this;
		}

		// 增加代码结束

		public Builder spoolDirectory(File directory) {
			this.spoolDirectory = directory;
			return this;
		}

		public Builder completedSuffix(String completedSuffix) {
			this.completedSuffix = completedSuffix;
			return this;
		}

		public Builder ignorePattern(String ignorePattern) {
			this.ignorePattern = ignorePattern;
			return this;
		}

		public Builder trackerDirPath(String trackerDirPath) {
			this.trackerDirPath = trackerDirPath;
			return this;
		}

		public Builder annotateFileName(Boolean annotateFileName) {
			this.annotateFileName = annotateFileName;
			return this;
		}

		public Builder fileNameHeader(String fileNameHeader) {
			this.fileNameHeader = fileNameHeader;
			return this;
		}

		public Builder annotateBaseName(Boolean annotateBaseName) {
			this.annotateBaseName = annotateBaseName;
			return this;
		}

		public Builder baseNameHeader(String baseNameHeader) {
			this.baseNameHeader = baseNameHeader;
			return this;
		}

		public Builder deserializerType(String deserializerType) {
			this.deserializerType = deserializerType;
			return this;
		}

		public Builder deserializerContext(Context deserializerContext) {
			this.deserializerContext = deserializerContext;
			return this;
		}

		public Builder deletePolicy(String deletePolicy) {
			this.deletePolicy = deletePolicy;
			return this;
		}

		public Builder inputCharset(String inputCharset) {
			this.inputCharset = inputCharset;
			return this;
		}

		public Builder decodeErrorPolicy(DecodeErrorPolicy decodeErrorPolicy) {
			this.decodeErrorPolicy = decodeErrorPolicy;
			return this;
		}

		public ReliableSpoolingFileEventExtReader build() throws IOException {
			return new ReliableSpoolingFileEventExtReader(spoolDirectory,
					completedSuffix, ignorePattern, trackerDirPath,
					annotateFileName, fileNameHeader, annotateBaseName,
					baseNameHeader, deserializerType, deserializerContext,
					deletePolicy, inputCharset, decodeErrorPolicy,
					annotateFileNameExtractor, fileNameExtractorHeader,
					fileNameExtractorPattern, convertToTimestamp,
					dateTimeFormat, splitFileName, splitBy, splitBaseNameHeader);
		}
	}

}
```


```java
/*
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements. See the NOTICE file distributed with this
 * work for additional information regarding copyright ownership. The ASF
 * licenses this file to you under the Apache License, Version 2.0 (the
 * "License"); you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
 * WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
 * License for the specific language governing permissions and limitations under
 * the License.
 */

package com.besttone.flume;

import org.apache.flume.serialization.DecodeErrorPolicy;

public class SpoolDirectorySourceConfigurationExtConstants {
	/** Directory where files are deposited. */
	public static final String SPOOL_DIRECTORY = "spoolDir";

	/** Suffix appended to files when they are finished being sent. */
	public static final String SPOOLED_FILE_SUFFIX = "fileSuffix";
	public static final String DEFAULT_SPOOLED_FILE_SUFFIX = ".COMPLETED";

	/** Header in which to put absolute path filename. */
	public static final String FILENAME_HEADER_KEY = "fileHeaderKey";
	public static final String DEFAULT_FILENAME_HEADER_KEY = "file";

	/** Whether to include absolute path filename in a header. */
	public static final String FILENAME_HEADER = "fileHeader";
	public static final boolean DEFAULT_FILE_HEADER = false;

	/** Header in which to put the basename of file. */
	public static final String BASENAME_HEADER_KEY = "basenameHeaderKey";
	public static final String DEFAULT_BASENAME_HEADER_KEY = "basename";

	/** Whether to include the basename of a file in a header. */
	public static final String BASENAME_HEADER = "basenameHeader";
	public static final boolean DEFAULT_BASENAME_HEADER = false;

	/** What size to batch with before sending to ChannelProcessor. */
	public static final String BATCH_SIZE = "batchSize";
	public static final int DEFAULT_BATCH_SIZE = 100;

	/** Maximum number of lines to buffer between commits. */
	@Deprecated
	public static final String BUFFER_MAX_LINES = "bufferMaxLines";
	@Deprecated
	public static final int DEFAULT_BUFFER_MAX_LINES = 100;

	/** Maximum length of line (in characters) in buffer between commits. */
	@Deprecated
	public static final String BUFFER_MAX_LINE_LENGTH = "bufferMaxLineLength";
	@Deprecated
	public static final int DEFAULT_BUFFER_MAX_LINE_LENGTH = 5000;

	/** Pattern of files to ignore */
	public static final String IGNORE_PAT = "ignorePattern";
	public static final String DEFAULT_IGNORE_PAT = "^$"; // no effect

	/** Directory to store metadata about files being processed */
	public static final String TRACKER_DIR = "trackerDir";
	public static final String DEFAULT_TRACKER_DIR = ".flumespool";

	/** Deserializer to use to parse the file data into Flume Events */
	public static final String DESERIALIZER = "deserializer";
	public static final String DEFAULT_DESERIALIZER = "LINE";

	public static final String DELETE_POLICY = "deletePolicy";
	public static final String DEFAULT_DELETE_POLICY = "never";

	/** Character set used when reading the input. */
	public static final String INPUT_CHARSET = "inputCharset";
	public static final String DEFAULT_INPUT_CHARSET = "UTF-8";

	/** What to do when there is a character set decoding error. */
	public static final String DECODE_ERROR_POLICY = "decodeErrorPolicy";
	public static final String DEFAULT_DECODE_ERROR_POLICY = DecodeErrorPolicy.FAIL
			.name();

	public static final String MAX_BACKOFF = "maxBackoff";

	public static final Integer DEFAULT_MAX_BACKOFF = 4000;

	// 增加代码开始
	public static final String FILENAME_EXTRACTOR = "fileExtractor";
	public static final boolean DEFAULT_FILENAME_EXTRACTOR = false;

	public static final String FILENAME_EXTRACTOR_HEADER_KEY = "fileExtractorHeaderKey";
	public static final String DEFAULT_FILENAME_EXTRACTOR_HEADER_KEY = "fileExtractorHeader";

	public static final String FILENAME_EXTRACTOR_PATTERN = "fileExtractorPattern";
	public static final String DEFAULT_FILENAME_EXTRACTOR_PATTERN = "(.)*";

	public static final String FILENAME_EXTRACTOR_CONVERT_TO_TIMESTAMP = "convertToTimestamp";
	public static final boolean DEFAULT_FILENAME_EXTRACTOR_CONVERT_TO_TIMESTAMP = false;

	public static final String FILENAME_EXTRACTOR_DATETIME_FORMAT = "dateTimeFormat";
	public static final String DEFAULT_FILENAME_EXTRACTOR_DATETIME_FORMAT = "yyyy-MM-dd";


	public static final String SPLIT_FILENAME = "splitFileName";
	public static final boolean DEFAULT_SPLIT_FILENAME = false;

	public static final String SPLITY_BY = "splitBy";
	public static final String DEFAULT_SPLITY_BY = "\\.";

	public static final String SPLIT_BASENAME_HEADER = "splitBaseNameHeader";
	public static final String DEFAULT_SPLIT_BASENAME_HEADER = "fileNameSplit";
	// 增加代码结束

}

```


```java
package com.besttone.flume;

import static com.besttone.flume.SpoolDirectorySourceConfigurationExtConstants.*;


import java.io.File;
import java.io.IOException;
import java.util.List;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

import org.apache.flume.ChannelException;
import org.apache.flume.Context;
import org.apache.flume.Event;
import org.apache.flume.EventDrivenSource;
import org.apache.flume.FlumeException;
import org.apache.flume.conf.Configurable;
import org.apache.flume.instrumentation.SourceCounter;
import org.apache.flume.serialization.DecodeErrorPolicy;
import org.apache.flume.serialization.LineDeserializer;
import org.apache.flume.source.AbstractSource;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.google.common.annotations.VisibleForTesting;
import com.google.common.base.Preconditions;
import com.google.common.base.Throwables;

public class SpoolDirectoryExtSource extends AbstractSource implements
		Configurable, EventDrivenSource {

	private static final Logger logger = LoggerFactory
			.getLogger(SpoolDirectoryExtSource.class);

	// Delay used when polling for new files
	private static final int POLL_DELAY_MS = 500;

	/* Config options */
	private String completedSuffix;
	private String spoolDirectory;
	private boolean fileHeader;
	private String fileHeaderKey;
	private boolean basenameHeader;
	private String basenameHeaderKey;
	private int batchSize;
	private String ignorePattern;
	private String trackerDirPath;
	private String deserializerType;
	private Context deserializerContext;
	private String deletePolicy;
	private String inputCharset;
	private DecodeErrorPolicy decodeErrorPolicy;
	private volatile boolean hasFatalError = false;

	private SourceCounter sourceCounter;
	ReliableSpoolingFileEventExtReader reader;
	private ScheduledExecutorService executor;
	private boolean backoff = true;
	private boolean hitChannelException = false;
	private int maxBackoff;

	// 增加代码开始

	private Boolean annotateFileNameExtractor;
	private String fileNameExtractorHeader;
	private String fileNameExtractorPattern;
	private Boolean convertToTimestamp;

	private String dateTimeFormat;
	
	private boolean splitFileName;
	private  String splitBy;
	private  String splitBaseNameHeader;

	// 增加代码结束

	@Override
	public synchronized void start() {
		logger.info("SpoolDirectorySource source starting with directory: {}",
				spoolDirectory);

		executor = Executors.newSingleThreadScheduledExecutor();

		File directory = new File(spoolDirectory);
		try {
			reader = new ReliableSpoolingFileEventExtReader.Builder()
					.spoolDirectory(directory).completedSuffix(completedSuffix)
					.ignorePattern(ignorePattern)
					.trackerDirPath(trackerDirPath)
					.annotateFileName(fileHeader).fileNameHeader(fileHeaderKey)
					.annotateBaseName(basenameHeader)
					.baseNameHeader(basenameHeaderKey)
					.deserializerType(deserializerType)
					.deserializerContext(deserializerContext)
					.deletePolicy(deletePolicy).inputCharset(inputCharset)
					.decodeErrorPolicy(decodeErrorPolicy)
					.annotateFileNameExtractor(annotateFileNameExtractor)
					.fileNameExtractorHeader(fileNameExtractorHeader)
					.fileNameExtractorPattern(fileNameExtractorPattern)
					.convertToTimestamp(convertToTimestamp)
					.dateTimeFormat(dateTimeFormat)
					.splitFileName(splitFileName).splitBy(splitBy)
					.splitBaseNameHeader(splitBaseNameHeader).build();
		} catch (IOException ioe) {
			throw new FlumeException(
					"Error instantiating spooling event parser", ioe);
		}

		Runnable runner = new SpoolDirectoryRunnable(reader, sourceCounter);
		executor.scheduleWithFixedDelay(runner, 0, POLL_DELAY_MS,
				TimeUnit.MILLISECONDS);

		super.start();
		logger.debug("SpoolDirectorySource source started");
		sourceCounter.start();
	}

	@Override
	public synchronized void stop() {
		executor.shutdown();
		try {
			executor.awaitTermination(10L, TimeUnit.SECONDS);
		} catch (InterruptedException ex) {
			logger.info("Interrupted while awaiting termination", ex);
		}
		executor.shutdownNow();

		super.stop();
		sourceCounter.stop();
		logger.info("SpoolDir source {} stopped. Metrics: {}", getName(),
				sourceCounter);
	}

	@Override
	public String toString() {
		return "Spool Directory source " + getName() + ": { spoolDir: "
				+ spoolDirectory + " }";
	}

	@Override
	public synchronized void configure(Context context) {
		spoolDirectory = context.getString(SPOOL_DIRECTORY);
		Preconditions.checkState(spoolDirectory != null,
				"Configuration must specify a spooling directory");

		completedSuffix = context.getString(SPOOLED_FILE_SUFFIX,
				DEFAULT_SPOOLED_FILE_SUFFIX);
		deletePolicy = context.getString(DELETE_POLICY, DEFAULT_DELETE_POLICY);
		fileHeader = context.getBoolean(FILENAME_HEADER, DEFAULT_FILE_HEADER);
		fileHeaderKey = context.getString(FILENAME_HEADER_KEY,
				DEFAULT_FILENAME_HEADER_KEY);
		basenameHeader = context.getBoolean(BASENAME_HEADER,
				DEFAULT_BASENAME_HEADER);
		basenameHeaderKey = context.getString(BASENAME_HEADER_KEY,
				DEFAULT_BASENAME_HEADER_KEY);
		batchSize = context.getInteger(BATCH_SIZE, DEFAULT_BATCH_SIZE);
		inputCharset = context.getString(INPUT_CHARSET, DEFAULT_INPUT_CHARSET);
		decodeErrorPolicy = DecodeErrorPolicy
				.valueOf(context.getString(DECODE_ERROR_POLICY,
						DEFAULT_DECODE_ERROR_POLICY).toUpperCase());

		ignorePattern = context.getString(IGNORE_PAT, DEFAULT_IGNORE_PAT);
		trackerDirPath = context.getString(TRACKER_DIR, DEFAULT_TRACKER_DIR);

		deserializerType = context
				.getString(DESERIALIZER, DEFAULT_DESERIALIZER);
		deserializerContext = new Context(context.getSubProperties(DESERIALIZER
				+ "."));

		// "Hack" to support backwards compatibility with previous generation of
		// spooling directory source, which did not support deserializers
		Integer bufferMaxLineLength = context
				.getInteger(BUFFER_MAX_LINE_LENGTH);
		if (bufferMaxLineLength != null && deserializerType != null
				&& deserializerType.equalsIgnoreCase(DEFAULT_DESERIALIZER)) {
			deserializerContext.put(LineDeserializer.MAXLINE_KEY,
					bufferMaxLineLength.toString());
		}

		maxBackoff = context.getInteger(MAX_BACKOFF, DEFAULT_MAX_BACKOFF);
		if (sourceCounter == null) {
			sourceCounter = new SourceCounter(getName());
		}
		
		//增加代码开始

		annotateFileNameExtractor=context.getBoolean(FILENAME_EXTRACTOR, DEFAULT_FILENAME_EXTRACTOR);
		fileNameExtractorHeader=context.getString(FILENAME_EXTRACTOR_HEADER_KEY, DEFAULT_FILENAME_EXTRACTOR_HEADER_KEY);
		fileNameExtractorPattern=context.getString(FILENAME_EXTRACTOR_PATTERN,DEFAULT_FILENAME_EXTRACTOR_PATTERN);
		convertToTimestamp=context.getBoolean(FILENAME_EXTRACTOR_CONVERT_TO_TIMESTAMP, DEFAULT_FILENAME_EXTRACTOR_CONVERT_TO_TIMESTAMP);
		dateTimeFormat=context.getString(FILENAME_EXTRACTOR_DATETIME_FORMAT, DEFAULT_FILENAME_EXTRACTOR_DATETIME_FORMAT);
		
		splitFileName=context.getBoolean(SPLIT_FILENAME, DEFAULT_SPLIT_FILENAME);
		splitBy=context.getString(SPLITY_BY, DEFAULT_SPLITY_BY);
		splitBaseNameHeader=context.getString(SPLIT_BASENAME_HEADER, DEFAULT_SPLIT_BASENAME_HEADER);
		
		
		
		//增加代码结束 
	}

	@VisibleForTesting
	protected boolean hasFatalError() {
		return hasFatalError;
	}

	/**
	 * The class always backs off, this exists only so that we can test without
	 * taking a really long time.
	 * 
	 * @param backoff
	 *            - whether the source should backoff if the channel is full
	 */
	@VisibleForTesting
	protected void setBackOff(boolean backoff) {
		this.backoff = backoff;
	}

	@VisibleForTesting
	protected boolean hitChannelException() {
		return hitChannelException;
	}

	@VisibleForTesting
	protected SourceCounter getSourceCounter() {
		return sourceCounter;
	}

	private class SpoolDirectoryRunnable implements Runnable {
		private ReliableSpoolingFileEventExtReader reader;
		private SourceCounter sourceCounter;

		public SpoolDirectoryRunnable(
				ReliableSpoolingFileEventExtReader reader,
				SourceCounter sourceCounter) {
			this.reader = reader;
			this.sourceCounter = sourceCounter;
		}

		@Override
		public void run() {
			int backoffInterval = 250;
			try {
				while (!Thread.interrupted()) {
					List<Event> events = reader.readEvents(batchSize);
					if (events.isEmpty()) {
						break;
					}
					sourceCounter.addToEventReceivedCount(events.size());
					sourceCounter.incrementAppendBatchReceivedCount();

					try {
						getChannelProcessor().processEventBatch(events);
						reader.commit();
					} catch (ChannelException ex) {
						logger.warn("The channel is full, and cannot write data now. The "
								+ "source will try again after "
								+ String.valueOf(backoffInterval)
								+ " milliseconds");
						hitChannelException = true;
						if (backoff) {
							TimeUnit.MILLISECONDS.sleep(backoffInterval);
							backoffInterval = backoffInterval << 1;
							backoffInterval = backoffInterval >= maxBackoff ? maxBackoff
									: backoffInterval;
						}
						continue;
					}
					backoffInterval = 250;
					sourceCounter.addToEventAcceptedCount(events.size());
					sourceCounter.incrementAppendBatchAcceptedCount();
				}
				logger.info("Spooling Directory Source runner has shutdown.");
			} catch (Throwable t) {
				logger.error(
						"FATAL: "
								+ SpoolDirectoryExtSource.this.toString()
								+ ": "
								+ "Uncaught exception in SpoolDirectorySource thread. "
								+ "Restart or reconfigure Flume to continue processing.",
						t);
				hasFatalError = true;
				Throwables.propagate(t);
			}
		}
	}
}

```

主要提供了如下几个设置参数：

```properties
tier1.sources.source1.type=com.besttone.flume.SpoolDirectoryExtSource   #写类的全路径名
tier1.sources.source1.spoolDir=/opt/logs   #监控的目录
tier1.sources.source1.splitFileName=true   #是否分隔文件名，并把分割后的内容添加到header中，默认false
tier1.sources.source1.splitBy=\\.                   #以什么符号分隔，默认是"."分隔
tier1.sources.source1.splitBaseNameHeader=fileNameSplit  #分割后写入header的key的前缀，比如a.log.2014-07-31,按“."分隔，则header中有fileNameSplit0=a,fileNameSplit1=log,fileNameSplit2=2014-07-31
```


（其中还有扩展一个通过正则表达式抽取文件名的功能也在里面，我们这里不用到，就不介绍了）

扩展了这个source之后，前面的那个需求就很容易实现了，只需要：

`tier1.sinks.sink1.hdfs.path=hdfs://master68:8020/flume/events/%{fileNameSplit0}/%{fileNameSplit2}`

a.log.2014-07-31这个文件的内容就会保存到hdfs://master68:8020/flume/events/a/2014-07-31目录下面去了。



接下来我们说说如何部署这个我们扩展的自定义的spooling directory source（基于CM的设置）。

首先，我们把上面三个类打成JAR包：SpoolDirectoryExtSource.jar
CM的flume插件目录为：`/var/lib/flume-ng/plugins.d`

然后再你需要使用这个source的agent上的/var/lib/flume-ng/plugins.d目录下面创建SpoolDirectoryExtSource目录以及子目录lib,libext,native,lib是放插件JAR的目录，libext是放插件的依赖JAR的目录，native放使用到的原生库（如果有用到的话）。

我们这里没有使用到其他的依赖，于是就把SpoolDirectoryExtSource.jar放到lib目录下就好了，最终的目录结构：


```properties
plugins.d/
plugins.d/SpoolDirectoryExtSource/
plugins.d/SpoolDirectoryExtSource/lib/SpoolDirectoryExtSource.jar
plugins.d/SpoolDirectoryExtSource/libext/
plugins.d/SpoolDirectoryExtSource/native/
```

重新启动flume agent,flume就会自动装载我们的插件，这样在flume.conf中就可以使用全路径类名配置type属性了。
最终flume.conf配置如下：


```properties
tier1.sources=source1
tier1.channels=channel1
tier1.sinks=sink1
tier1.sources.source1.type=com.besttone.flume.SpoolDirectoryExtSource
tier1.sources.source1.spoolDir=/opt/logs
tier1.sources.source1.splitFileName=true
tier1.sources.source1.splitBy=\\.
tier1.sources.source1.splitBaseNameHeader=fileNameSplit
tier1.sources.source1.channels=channel1
tier1.sinks.sink1.type=hdfs
tier1.sinks.sink1.channel=channel1
tier1.sinks.sink1.hdfs.path=hdfs://master68:8020/flume/events/%{fileNameSplit0}/%{fileNameSplit2}
tier1.sinks.sink1.hdfs.round=true
tier1.sinks.sink1.hdfs.roundValue=10
tier1.sinks.sink1.hdfs.roundUnit=minute
tier1.sinks.sink1.hdfs.fileType=DataStream
tier1.sinks.sink1.hdfs.writeFormat=Text
tier1.sinks.sink1.hdfs.rollInterval=0
tier1.sinks.sink1.hdfs.rollSize=10240
tier1.sinks.sink1.hdfs.rollCount=0
tier1.sinks.sink1.hdfs.idleTimeout=60
tier1.channels.channel1.type=memory
tier1.channels.channel1.capacity=10000
tier1.channels.channel1.transactionCapacity=1000
tier1.channels.channel1.keep-alive=30
```

附上一张用logger作为sink的查看日志文件的截图：
[![flume-source](https://i0.wp.com/i.loli.net/2019/12/31/mXZvLaxSMVFkyj3.png)](https://i0.wp.com/i.loli.net/2019/12/31/mXZvLaxSMVFkyj3.png)